# MemTable

成员变量

- comparator_，InternalKey 的 comparator
- refs_，当前 memtable 被引用数，可能被多个线程共享
- arena_，内存分配器
- table_，是一个 skiplist 对象

![](./img/keyvalformat.png)


**插入**
其中 memtable 的跳表中保存的是 keyvalue 编码的结果，并没有将 keyvalue 分开存储。

它提供了一个迭代器实现了对 skiplist 的各种操作，实际上调用了 skiplist 本身的迭代器

**查找**
memtable 查找时通过一个 LookupKey 进行查找

如果 memtable 写满或者写到一个阈值，那么就变成 immutable memtable，后台启动一个线程将其写入磁盘。`DBImpl::WriteLevel0Table` 函数会调用 TableBuilder 将 MemTable 写成 SST 