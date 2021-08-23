由于rocksdb 的append写特性数据本身保存了多个版本，所以实现snapshot比较简单。
class Snapshot {
 public:
  // returns Snapshot's sequence number
  virtual SequenceNumber GetSequenceNumber() const = 0;

 protected:
  virtual ~Snapshot();
};

从snapshot类可以看到， rocksdb snapshot只需要关心打快照时候的key对应的SequenceNumber。由于是全觉单调递增的，所以这个也是一个唯一标示。rocksdb的快照是一个只有内存态保持的状态，在数据库重启之后快照会丢失。
在记录了这个SequenceNumber之后，整个db会维护一个快照双向链表，链表元素在SnapshotImpl。 
class SnapshotImpl : public Snapshot {
 public:
  SequenceNumber number_;  // const after creation

  virtual SequenceNumber GetSequenceNumber() const override { return number_; }

 private:
  friend class SnapshotList;

  // SnapshotImpl is kept in a doubly-linked circular list
  SnapshotImpl* prev_;
  SnapshotImpl* next_;

  SnapshotList* list_;                 // just for sanity checks

  int64_t unix_time_;

  // Will this snapshot be used by a Transaction to do write-conflict checking?
  bool is_write_conflict_boundary_;
};

rocksdb在做compaction时候会检查快照双向链表，对于有快照的SequenceNumber小于的sst文件做删除操作时候会单独做保留处理。快照在wal到sst文件时候就会生成，可以看BuildTable函数。详细处理流程可以看compation和写sst流程。
