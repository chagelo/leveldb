```bash
DBImpl::Get
    mem->Get
    imm->Get
    current(Version *)->Get
        ForEachOverlapping
            State::Match
                state->vset->table_cache_(TableCache)->Get
                    TableCache::FindTable
                        Table::Open
                    t(Table)->InternalGet
                        iiter(Iterator index_block_->NewIterator)->Seek(k);
                        !filter->KeyMayMatch
                        BlockReader
                            block_cache->Lookup(key)
                            ReadBlock
                            block_cache->Insert
                        block_iter->Seek(k)
    MaybeScheduleCompaction();

```


1. mem->Get
2. imm->Get
3. current->Get，Version 保存当前的 SST files 的一些状态，哪些 level 有哪些 sst files，其中每个 file 的最大 key 最小 key。所以必然需要持久化这个 Version
4. ForEachOverlapping，在 level 0 search，key 介于最大最小之间，然后块内找；在 level i>=1，二分找第一个 largest key >= target key 的块，然后块内线性找
5. State::Match，当通过 Version 中的 FileMetaData，确定到要查找的 file，回调调用这个函数
6. state->vset->table_cache_->Get，这个时候去查找 TableCache
7. TableCache::FindTable，调用这个函数的目的就是想要读到 TableCache 中对应 file 的 metadata
8. Table::Open，从磁盘读取到 metadata，并将这样的指向这块 metadata 的 pointer 插入到 TableCache
9. t(Table)->InternalGet
10. iiter->Seek(k)，在 index_block 内根据 restart point 二分查找，到具体的 datablock，但 datablock 不一定在内存里
11. !filter->KeyMayMatch，指定 datablock，布隆过滤器找，不存在直接 return
12. BlockReader，有了 datablock 的 handle，去构造 datablcok 的 iterator，这个函数返回的就是 iterator；如果 block 不在内存里，要读进来然后插入到 block cache 中
  - 有个疑惑的地方，我们有很多次查找，但不一定每次都找到的 datablock 一定就是 key 所在的，那么有必要插入进 lru cache 中吗？*貌似插入进去的不是[user_key, datablock handler]*，好像是 datablock 的 restart point 的值
13. block_iter->Seek(k)，通过迭代器二分找准确的值
14. 在查找过程中，某个块可能 miss 次数过多，这是一种触发 compaction 的方式
