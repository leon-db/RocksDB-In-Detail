#数据更新

##1. 引擎中的写操作
  通常我们说更新，一般来说包含:
* 插入/新增
* 更新
* 删除

在数据库中更新操作在设计上往往有很多地方需要考虑。比如说性能、并发控制、数据隔离性等。
在RocksDB中因为设计上就是把多个随机写转顺序写，同时文件是追加的方式减少了不必要的锁开销。
正是因为这种抽象，所以需要对写进行一个抽象。这种抽象的方式也很简单，定义多种写
操作模式，然后以追加的方式来抽象所有的写入操作。

比如说：更新一个值，就是插入一个新的值。老的值通过版本来过滤。删除一个值，同样是通过追加写一个
删除的标记来做到删除的语义。

### 糟糕的更新性能
  RocksDB本身的一个设计思想就是利于写，同时保证读性能可控就好。然后我们很多操作是
依赖于写的。也就是说如果读性能不好，在这种情况下也会影响写性能。
  一个显而易见的例子就是计数类的需求。比如说有一个count，我们现在要做count++操作。
分解的步骤为：
    1. 先读取count在存储引擎中的值
    2. 在内存中对count++
    3. 把更新后的count值回写存储引擎

为了解决这个读影响写性能问题，RocksDB引擎引入一种机制：Merge。
这种方案的做法也比较简单直接，对于每次递增操作，只把这个递增的语义写入, 然后在读取的时候
把所有的语义merge在一起。这种方案的优点是：写入更新不在依赖读取的性能，缺点也很明显：本来读的性能
就偏差，这会进一步的弱化读取性能。对于读性能的控制，目前主要是通过compaction来优化。详细可以参考后面
关于compaction系列的文章

## 写接口
通过前面的分析，我们在来看看RocksDB有哪些接口是关于数据写操作的，详细如下：
1. Put
2. Merge
3. Delete
4. Write

注意：关于SingleDelete这样的其实也是删除，只是在性能上的考虑，对写的主流程分析不构成影响。
这块后面的相关的文章会详细的介绍

上面四种写，前面的Put, Merge, Delete写操作最终都会调用Write来做实际的写入操作。唯一的区别就是
类型的不同。

Put操作：
```c++
Status DB::Put(const WriteOptions& opt, ColumnFamilyHandle* column_family,
const Slice& key, const Slice& value) {
  ...
  const Slice* ts = opt.timestamp;
  WriteBatch batch(key.size() + ts_sz + value.size() + 24, /*max_bytes=*/0,
                   ts_sz);
  Status s = batch.Put(column_family, key, value);
  if (!s.ok()) {
    return s;
  }
  s = batch.AssignTimestamp(*ts);
  if (!s.ok()) {
    return s;
  }
  return Write(opt, &batch);
}
```
上面的逻辑也很简单, 根据写入的参数，构建write_batch然后调用Write接口写入。

Merge操作:

```c++
  WriteBatch batch;
  Status s = batch.Merge(column_family, key, value);
  if (!s.ok()) {
    return s;
  }
  return Write(opt, &batch);
```

对于Delete操作和Merge操作及Put操作一样，都是构建本次写入的write_batch。
正如前面所述，因为对写入做了特殊的抽象，这个抽象就是这里write_batch里面保存的key的类型
可能不同而实现不同的写入逻辑。到此我们可以有一个直观的认识，
其实所有的写入操作都是通过构建不同类型的key,放入到writebatch中，然后直接调用RocksDB的接口即可.

##2. 写操作实现过程
   上面大致总结了RocksDB中写入操作的一个基本抽象，在了解详细的写入操作实现之前，
我们先来想一下写操作的实现有哪些因素是要考虑的：

1. 写操作要实现怎样的语义
2. 多个读写操作如何并发，以及有怎样的影响
3. 常规隔离级别下，读写并发正确性是怎样保证的
4. 在使用层面，写入有怎样的参数，在实践中如何配置

如果你有上面的这些疑问，那么不妨我们深入的了解接下来的介绍。
首先，对于写入的语义，上面的章节已经做了详细的介绍了。对于问题2、3、4详细参考Write实现

### 1. WriteOptions
```c++
  virtual Status Write(const WriteOptions& options, WriteBatch* updates) = 0;
```   
可以看到每个写入操作最主要的就是：
1. 要指定本次写入要控制的参数
2. 本次写入要写入的kv数据，RocksDB都封装在write batch中

关于WriteOptions主要有下面的写入控制参数
```c++
struct WriteOptions {
  // 默认 false
  // 主要控制WAL要不要sync落盘，不然就只保存在操作系统的pagecache中
  // 这在机器某些情况下会丢失数据，比如说机器死机等
  bool sync;   
  // 默认false
  // 不写WAL数据，有丢失数据风险，数据只写memtable
  bool disableWAL;
  // 默认false
  // 如果设置了，那么当写入时候，当写入的数据不存在对应的column family
  // 的时候，不会报错。
  bool ignore_missing_column_families;
  // 默认false
  // 如果设置当我们碰到流控要等待的时候会直接返回错误，而不是等待
  bool no_slowdown;
  // 默认false
  bool low_pri;
  //  默认false
  bool memtable_insert_hint_per_batch;
  // 默认nullptr
  const Slice* timestamp;
};
```
  