对于block块的每一个数据块构建方式用到的核心数据如下：

```c++
   const int block_restart_interval_;
   const bool use_delta_encoding_;
   
   std::string buffer_;               // 数据块kv缓存
   std::vector<uint32_t> restarts_;   // 重启点
   size_t estimate_;                  // 预估的字节数
   int counter_;                      // 相对重启点的逻辑偏移(相对重启点第几个kv对）
   bool finished_;                    // 是否标记完成block块的
   std::string last_key_;             // 因为要做共享前缀共用编码，因此需要记录下上一个key(更新重启点的那个key)
```
正如变量的修饰是const那样，这两个变量主要是反应block的配置信息。
  1.block_restart_interval_   主要是一种在磁盘效率和查询效率之间做一种trade-off, 超过多少个之后重置一下,这样可以加速查询速度
  2.use_delta_encoding_       是否使用共享前缀的编码方式，默认为true, 这样可以节约磁盘空间

因为sst文件的是append only的方式创建，因此这块在接口上就只有三个接口：
  1. Reset  把内存变量重置一下，继续使用
  2. Add    把一个kv对添加到block块中
  3. Finish 当达到block块的大小的时候，调用这个函数把缓存在内存中的buffer_ 标记为finish可以落盘了

# 往block块插入数据
  
##编码方式
```c++
    <shared><non_shared><value_size>
```

需要说明的是：shared和non_shared的这些概念中，参与的对象是哪些？
每个key都和本组的前一个key进行比较。直到重启点.

因此在这种模式下就很简单了：
    <shared><non_shared><value_size><key_except_shared><value>
要完整的理解这个编码必须要结合重启点来看， 我们举一个例子：
     1. aaaa:11111
     2. aaab:22222
     3. aacb:33333
对于这样的两个key， 我们的编码为：
     1. <0><4><5><aaaa><11111>     // 重启点
     2. <3><1><5><b><22222>
     3. <2><2><5><cb><33333>
显然如果不使用delta编码那么就是：
     1. <0><4><5><aaaa><11111>
     2. <0><4><5><aaab><22222>
     3. <0><4><5><aacb><33333>

当我们插入的kv超过了一个组的时候，会记录新的重启点信息，并且更新last_key


因为使用了shared编码方式，所以当读取的时候要完整的解析出一个完整的key必须要结合重启点数据才可以完整的解析出来。
为了高效的定位，RocksDB会记录这个重启点的在块中的一个偏移地址（字节）

第一个重启点就是：偏移地址为0, 整个重启点可以理解为一个 【）左闭又开的区间
当插入key导致一个组超过数量的时候，就会记录之前的buffer_的字节数作为本组的偏移，并且把counter_重置. 

当一个block块调用Finish的时候，要做的事情就是把之前的编码的数据和重启点一起flush到磁盘中去。
磁盘布局如下：

```
组1      <shared><non_shared><value_size><key1><value1>
         <shared><non_shared><value_size><key2><value2>
         ...
         ...
组k      <shared><non_shared><value_size><key3><value3>
         <shared><non_shared><value_size><key4><value4>
组n      <shared><non_shared><value_size><key5><value5>
restart  (组的首地址偏移地址)
restart num (组数)
```


