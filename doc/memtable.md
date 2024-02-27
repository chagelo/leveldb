# MemTable

成员变量

- comparator_，InternalKey 的 comparator
- refs_，当前 memtable 被引用数，可能被多个线程共享
- arena_，内存分配器
- table_，是一个 skiplist 对象

![](./img/keyvalformat.png)


（这里写错了 keysize 应该换成 internal_key_size）

**插入**
其中 memtable 的跳表中保存的是 keyvalue 编码的结果，并没有将 keyvalue 分开存储。

它提供了一个迭代器实现了对 skiplist 的各种操作，实际上调用了 skiplist 本身的迭代器

**查找**
memtable 查找时通过一个 LookupKey 进行查找 (user_key | batch_seqnum | type)

## WriteLevel0Table

如果 memtable 写满或者写到一个阈值，那么就变成 immutable memtable，后台启动一个线程将其写入磁盘。`DBImpl::WriteLevel0Table` 函数会调用 TableBuilder 将 MemTable 写成 SST 

这个方法会在两个地方被调用

1. 从 WAL 恢复的时候
2. `DBImpl::CompactMemTable` 调用，该方法在两个地方被调用
  - 当在启动的后台进程进行 compaction 时，如果发现 imm_ 不空，则先调用该函数将 immutable memtable 写磁盘
  - immu 写磁盘和 compaction 操作都会对版本信息进行更改，这是互斥的，因此在 compaction 之前

`CompactMemTable` 中会对 MemTable 写 SST 的操作

```cpp
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();
  pending_outputs_.insert(meta.number);
  Iterator* iter = mem->NewIterator();
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long)meta.number);

  Status s;
  {
    mutex_.Unlock();
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long)meta.number, (unsigned long long)meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);

  // Note that if file_size is zero, the file has been deleted and
  // should not be added to the manifest.
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != nullptr) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    edit->AddFile(level, meta.number, meta.file_size, meta.smallest,
                  meta.largest);
  }

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);
  return s;
}
```

1. 调用 BuildTable 完成 MemTable 到 SST 的转换，每个 SST 会新创建一个文件 number，最后的 SST 的名称可以为 000012.ldb
2. 调用 PickLevelForMemTableOutput 为新的 SST 文件挑选 level，最低放到 level 0，最高放到 level 2。这里还会修改 VersionEdit，添加一个文件记录。包含文件的 number、size、最大和最小 key。
3. 更新 SST 插入的那层的信息


# PickLevelForMemTableOutput 

该方法将 Immutable memtable 向下 push，最多可以写到第二层

```cpp
int Version::PickLevelForMemTableOutput(const Slice& smallest_user_key,
                                        const Slice& largest_user_key) {
  int level = 0;
  // 如果和 level 0 有重叠就返回 level 0
  if (!OverlapInLevel(0, &smallest_user_key, &largest_user_key)) {
    // Push to next level if there is no overlap in next level,
    // and the #bytes overlapping in the level after that are limited.
    InternalKey start(smallest_user_key, kMaxSequenceNumber, kValueTypeForSeek);
    InternalKey limit(largest_user_key, 0, static_cast<ValueType>(0));
    std::vector<FileMetaData*> overlaps;
    while (level < config::kMaxMemCompactLevel) {
      if (OverlapInLevel(level + 1, &smallest_user_key, &largest_user_key)) {
        break;
      }
      if (level + 2 < config::kNumLevels) {
        // Check that file does not overlap too many grandparent bytes.
        // overlaps 返回有重叠部分的文件列表，然后计算文件的总大小
        GetOverlappingInputs(level + 2, &start, &limit, &overlaps);
        const int64_t sum = TotalFileSize(overlaps);
        if (sum > MaxGrandParentOverlapBytes(vset_->options_)) {
          break;
        }
      }
      level++;
    }
  }
  return level;
}
```

这个方法属于 Version 类，Version 类保存每一层含有哪些文件，每个 SST 的最大 Key 和最小 Key 又是什么

Immutable memtable 落盘最多可以写到 SST 的第 2 层

1. 和 level 0 有重叠就返回，如果一直没有重叠就可以一直往下
2. 如果 level i+1 没有重叠，和 level i+2 有重叠但比较小，则可以往下一层，继续判断


# MakeRoomForWrite

LevelDB每次写入key-value都是写入到内存中的Memtable中的，但是Memtable的空间不是无限的，Memtable写满后，就需要调用MakeRoomForWrite把Memtable转存为Immutable Memtable，并创建新的Memtable来存储写入数据。必要时还会调度后台线程把Immutable Memtable落盘，以及合并SST文件。

1. 如果允许 delay，并且 l0 层文件数量超过慢写阈值（默认 8 个文件），就等待 1ms，然后把 allow_delay 设置为 false，所以慢写延迟操作最多执行一次，避免上面的 Writer 主流程被阻塞太久
2. 