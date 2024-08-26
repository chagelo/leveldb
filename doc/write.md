# DBImpl::Writer

- status：存储写入的状态
- batch：指向要写入的数据
- sync：表示是否要把预写日志（Write Ahead Log, WAL）Sync到磁盘上
- done：表示Writer是否完成
- cv：对条件变量的封装

#  DBImpl::Write

```cpp
DBImpl::Put
    DB::Put
        batch.Put
        DBImpl::Write
            DBImpl::MakeRoomForWrite(updates == nullptr)
                mem_ = new MemTable
                s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);
                MaybeScheduleCompaction();
            DBImpl::BuildBatchGroup(&last_writer);
            log_->AddRecord(WriteBatchInternal::Contents(write_batch));
            logfile_->Sync();
            WriteBatchInternal::InsertInto(write_batch, mem_);
```

1. 设置当前 Writer 
2. 使用条件变量加互斥量实现，互斥访问 Writer_ 队列，当前 Writer 加入队列，如果当前 Writer 未完成且不是队首，那么就睡眠
3. 如果完成了或者是队首，就往下

levelDB 还对 WriteBatch 进行了合并，如果发现当前 Writer 是队首，且未完成写，则将它和队列后面的进行合并，

- 如果第一个 Writer 要求 sync 当前的 Writer 不要求 sync，那么就推出
- 如果合并后的 size 大于 max_size（1MB）就推出


1. DBImpl::MakeRoomForWrite，这个函数是用来确保，当用户的 batch 写时，内存里面还有空间，mem、imm，里面很多细节
  - 如果有需要写的内容，那么  allow_delay = true，并且如果此时 L0 files 数量触发慢写阈值
    - 释放锁
    - 睡眠 1s
    - 睡眠结束，allow_delay，当前写不能再次推迟
    - 上锁
  - 如果有需要写的内容，并且 mem 还有容量，否则下一步
  - imm 不为空，即 imm 正在写磁盘，mem 没空间了，那么等待后台 imm 写完成，否则下一步
  - mem 没空间，imm 为空，且 L0 文件太多触发停止写阈值，等待，否则下一步
  - mem 没空间，imm 为空，此时可以把 mem 转换成 imm，并在磁盘创建文件，为 imm 写磁盘做准备
2. s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile); 这个函数会为当前新的 mem 创建一个 000001.log wal 文件，注意此时 imm 是空的，所以不用关注 imm 对应的 wal
3. BuildBatchGroup，当前有一个 writers_ 是个 Writer task 写队列，这个函数会尽可能从对队首（当前处理的是队首 Writer task）往后合并 Writer
4. log_->AddRecord 会把当前和并之后的 Writer 里的 writer batch 里的数据拿出来写到 mem 对应的 wal 中，这个函数内部调用了 write 系统调用，如果开启 sync 标志，此时还会调用 fsync。wal 中会写一些校验信息，所以即使挂掉，也还 ok
5.  WriteBatchInternal::InsertInto(write_batch, mem_)，当上一步完成之后，会将数据写到 mem 中
  - 写 mem 可能出现冲突吗，可能出现不一致，但不会出现读写冲突，这个原因是 skiplist 中节点都是原子的单向指针；不会出现写写冲突，因为只有队首元素才能写 mem，写完才会调用 pop_back，
  - 在写 mem 和 wal 的过程中释放锁，使得其他线程可以向 queue 中 append writer task，或者做一些 compactin 的操作；读 mem 和 imm 和 sst files 不会上锁
6. 在这之后就是删除完成的 writer task，唤醒那些被阻塞的 task




# ref

1. [LevelDB源码解析(14) 写操作之Write主流程](https://www.huliujia.com/blog/24af576aa3/)
