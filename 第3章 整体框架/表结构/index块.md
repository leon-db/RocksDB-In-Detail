# IndexBuilder 构建index

IndexBlocks 索引的meta信息, 包含一个主的index索引块和可选的多个meta索引块信息
```c++
struct IndexBlocks {
  Slice index_block_contents;
  std::unordered_map<std::string, Slice> meta_blocks;
};
```

IndexBuilder 作为一个抽象的index类实现，正如IndexType这个枚举类型描述的那样，我们有三种类型的index.
因此在这个类中有个工厂方法用来创建不同的IndexBuilder子类实现.

所有的子类都要实现的核心方法：
     1.  AddIndexEntry
     2.  OnKeyAdded
     3.  Finish

