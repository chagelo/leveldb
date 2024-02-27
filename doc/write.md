# DBImpl::Writer

- status：存储写入的状态
- batch：指向要写入的数据
- sync：表示是否要把预写日志（Write Ahead Log, WAL）Sync到磁盘上
- done：表示Writer是否完成
- cv：对条件变量的封装

#  DBImpl::Write

```cpp
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
  Writer w(&mutex_);
  w.batch = updates;
  w.sync = options.sync;
  w.done = false;

  MutexLock l(&mutex_);
  writers_.push_back(&w);
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();
  }
  if (w.done) {
    return w.status;
  }

  // May temporarily unlock and wait.
  Status status = MakeRoomForWrite(updates == nullptr);
  uint64_t last_sequence = versions_->LastSequence();
  Writer* last_writer = &w;
  if (status.ok() && updates != nullptr) {  // nullptr batch is for compactions
    WriteBatch* write_batch = BuildBatchGroup(&last_writer);
    WriteBatchInternal::SetSequence(write_batch, last_sequence + 1);
    last_sequence += WriteBatchInternal::Count(write_batch);

    // Add to log and apply to memtable.  We can release the lock
    // during this phase since &w is currently responsible for logging
    // and protects against concurrent loggers and concurrent writes
    // into mem_.
    {
      mutex_.Unlock();
      status = log_->AddRecord(WriteBatchInternal::Contents(write_batch));
      bool sync_error = false;
      if (status.ok() && options.sync) {
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
      if (status.ok()) {
        status = WriteBatchInternal::InsertInto(write_batch, mem_);
      }
      mutex_.Lock();
      if (sync_error) {
        // The state of the log file is indeterminate: the log record we
        // just added may or may not show up when the DB is re-opened.
        // So we force the DB into a mode where all future writes fail.
        RecordBackgroundError(status);
      }
    }
    if (write_batch == tmp_batch_) tmp_batch_->Clear();

    versions_->SetLastSequence(last_sequence);
  }

  while (true) {
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status;
      ready->done = true;
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }

  // Notify new head of write queue
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }

  return status;
}
```

1. 设置当前 Writer 
2. 使用条件变量加互斥量实现，互斥访问 Writer_ 队列，当前 Writer 加入队列，如果当前 Writer 未完成且不是队首，那么就睡眠
3. 如果完成了或者是队首，就往下

levelDB 还对 WriteBatch 进行了合并，如果发现当前 Writer 是队首，且未完成写，则将它和队列后面的进行合并，

- 如果第一个 Writer 要求 sync 当前的 Writer 不要求 sync，那么就推出
- 如果合并后的 size 大于 max_size（1MB）就推出


# ref

1. [LevelDB源码解析(14) 写操作之Write主流程](https://www.huliujia.com/blog/24af576aa3/)
2. 