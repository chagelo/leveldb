# WriteBatch

一个 writebatch 的格式如下


8B|4B|变长|变长|...
-|-|-|-|-
seqnum|count|record|record|...

- seqnum，每个 writebatch 有一个唯一的序列号
- count，总共包含的 record 数目
- record，它的格式如下，注意这个 record 里面没有 序列号了，而 memtable 里是有序列号的


1B|变长|keysize|变长|valsize
-|-|-|-
Type|keysize|key|valsize|val

所有字段用一个字符串保存


**seqnum**
这个 writebatch 的构造函数里面只是初始化了一个空的字符串，并没有赋予 seqnum 初始值，这个初始值是由 `WriteBatchInternal::SetSequence` 方法赋予的。后续构造 memtable 插入的值会使用这个 seqnum，memtable 插入的 record 需要一个 seqnum

# WriteBatchInternal

这个类内提供了很多静态方法，对 writebatch 进行操作。比如设置 writebatch 的 seqnum、count、record 字段，获取 writebatch 的 seqnum、count、record 字段

`InsertInto` 构造了一个 MemtableInserter


# MemtableInserter

这个类用于将记录插入 MemTable

1. seqence_，序列号
2. mem_，memtable

# BuildBatchGroup

BuildBatchGroup 将写任务队列合并，减少磁盘写次数，提高写性能，虽然每次写都要向 memtable 写，但在那之前需要写 wal，所以需要写磁盘。

如果是串行写入，那么不会有合并操作，只有用 writebatch 并行写时才会出现任务合并