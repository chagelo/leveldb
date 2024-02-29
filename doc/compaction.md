# Compaction 类型

LevelDB中LSM-Tree的Compaction操作分为两类，分别是Minor Compaction与Major Compaction。

- Minor Compaction（Immutable MemTable -> SSTable）：将Immutable MemTable转储为level-0 SSTable写入。
- Major Compaction（Low-level SSTable -> High-level SSTable）：合并压缩第i层的SSTable，生成第i+1层的SSTable。

在LevelDB中，Major Compaction还可以按照触发条件分为三类：

- Size Compaction：根据每层总SSTable大小除以 level i 层总SST大小最大值触发（level-0根据SSTable数除以 kL0_CompactionTrigger=4）的Major Compaction。
- Seek Compaction：根据SSTable的seek miss触发的Major Compaction。
- Manual Compaction：LevelDB使用者通过接口void CompactRange(const Slice* begin, const Slice* end)手动触发。

## Compaction 优先级

Minor Compaction > Manual Compaction > Size Compaction > Seek Compaction

# 后台线程

后台线程是不存在并发的，同一时刻只会有一个后台线程在执行。后台线程和Write线程存在并发竞争，所以在关键区域要使用成员变量mutex_加锁。LevelDB 只使用了 1 个后台线程，因此 Compaction 仍是串行而不是并行的

调用顺序为
MaybeScheduleCompaction->Schedule

在 Schedule 内部主线程创建子线程然后子线程去执行队列队列中的任务，主线程将任务入队然后唤醒后台线程

# LevelDB SST 分层

LevelDB 每层的单个文件最大 size 都是一样的，通过限制不同层的文件总 size 来控制每层的文件数量，L1 层的最大总 size 为 1M，其他层都是低一层的 10 倍，默认的分层 size 如下表

L1|L2|L3|...
-|-|-|-
1M|10M|100M|1000M|...

L0 层是通过文件数量来限制的，涉及到几个不同的限制程度，达到的限制程度越高，对 Memtable 落盘影响越大，默认达到 4 个就进行合并，达到 12 个，就暂停 Memtable 落盘。


# DBImpl::BackgroundCompaction

```cpp
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  if (imm_ != nullptr) {
    CompactMemTable();
    return;
  }

  Compaction* c;
  bool is_manual = (manual_compaction_ != nullptr);
  InternalKey manual_end;
  if (is_manual) {
    ManualCompaction* m = manual_compaction_;
    c = versions_->CompactRange(m->level, m->begin, m->end);
    m->done = (c == nullptr);
    if (c != nullptr) {
      manual_end = c->input(0, c->num_input_files(0) - 1)->largest;
    }
    Log(options_.info_log,
        "Manual compaction at level-%d from %s .. %s; will stop at %s\n",
        m->level, (m->begin ? m->begin->DebugString().c_str() : "(begin)"),
        (m->end ? m->end->DebugString().c_str() : "(end)"),
        (m->done ? "(end)" : manual_end.DebugString().c_str()));
  } else {
    c = versions_->PickCompaction();
  }

  Status status;
  if (c == nullptr) {
    // Nothing to do
  } else if (!is_manual && c->IsTrivialMove()) {
    // Move file to next level
    assert(c->num_input_files(0) == 1);
    FileMetaData* f = c->input(0, 0);
    c->edit()->RemoveFile(c->level(), f->number);
    c->edit()->AddFile(c->level() + 1, f->number, f->file_size, f->smallest,
                       f->largest);
    status = versions_->LogAndApply(c->edit(), &mutex_);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    VersionSet::LevelSummaryStorage tmp;
    Log(options_.info_log, "Moved #%lld to level-%d %lld bytes %s: %s\n",
        static_cast<unsigned long long>(f->number), c->level() + 1,
        static_cast<unsigned long long>(f->file_size),
        status.ToString().c_str(), versions_->LevelSummary(&tmp));
  } else {
    CompactionState* compact = new CompactionState(c);
    status = DoCompactionWork(compact);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    CleanupCompaction(compact);
    c->ReleaseInputs();
    RemoveObsoleteFiles();
  }
  delete c;

  if (status.ok()) {
    // Done
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // Ignore compaction errors found during shutting down
  } else {
    Log(options_.info_log, "Compaction error: %s", status.ToString().c_str());
  }

  if (is_manual) {
    ManualCompaction* m = manual_compaction_;
    if (!status.ok()) {
      m->done = true;
    }
    if (!m->done) {
      // We only compacted part of the requested range.  Update *m
      // to the range that is left to be compacted.
      m->tmp_storage = manual_end;
      m->begin = &m->tmp_storage;
    }
    manual_compaction_ = nullptr;
  }
}
```

1. 如果 imm_ 不为空，则先将 imm_ 写 SST 的 l0-l2 层，然后返回
2. 看是否能够通过移动来进行合并（什么时候能够通过移动进行合并，当前层 level i 文件只有一个，下一层 level i+1 层文件 0 个，且这个文件和 level i+2 层重叠部分小于给定阈值）
3. 如果可以直接移动，并且更改 Version 的文件 metadata
4. 正常合并，调用 `DoCompactionWork`

### 选择 level i 层 需要合并的文件

`VersionSet::PickCompaction` 负责找到需要执行合并的 level

优先考虑 size_compaction，其次 seek_compaction，如果两个条件都不满足就返回不合并

**size_compaction**

如果发生 size_compaction 执行下面的逻辑

compact_pointer_[level] 保存了level层上次被合并的文件的 largest key，从 level 层文件中选出 largest_key 最小的文件，使其大于 compact_pointer_[level]，如果 compact_pointer_[level] 为空，或者level层没有文件的 larget_key 比 compact_pointe_[level] 大，就选择第一个文件，从头开始。这个机制保证了level层所有文件都有均等的机会被合并，避免了一直合并头部的文件，导致后面的文件没有机会被合并。

**seek_compaction**

如果发生 seek_compaction 执行下面的逻辑

逻辑简单直接，要合并的level是file_to_compact_level_，要合并的level层文件是file_to_compact_。这两个值是读操作的时候赋值的，当一个文件seek miss的次数超过阈值，就会把这个文件和所在层设置为需要合并。这里推测作者的想法是查询的key在这个文件范围内，但是却不在这个文件里，每次到这里都需要多读一次磁盘，那么把这个文件往高一层合并，可以避免在当前层无效地读文件。

**L0层特殊处理**
因为L0层的文件之间是有重叠的，所以会递归的把所有和选中文件有重叠的文件也加入到被合并的文件列表中。所谓的递归，是指如果有新文件加入到列表中，对新文件，也要递归的去查找和新文件有重叠的文件。

## 挑选 level i+1 层需要合并的

Compaction 的 inputs_ 是大小为 2 的 std::vector<FileMetaData*> 数组，下标 1 存储 level i 层， 下标 1 存储 level i+1 层 

`version_set.cc` 的 `SetupOtherInputs`挑选 level i+1 层需要合并的文件

```cpp
void VersionSet::SetupOtherInputs(Compaction* c) {
  const int level = c->level();
  InternalKey smallest, largest;

  AddBoundaryInputs(icmp_, current_->files_[level], &c->inputs_[0]);
  GetRange(c->inputs_[0], &smallest, &largest);

  current_->GetOverlappingInputs(level + 1, &smallest, &largest,
                                 &c->inputs_[1]);
  AddBoundaryInputs(icmp_, current_->files_[level + 1], &c->inputs_[1]);

  // Get entire range covered by compaction
  InternalKey all_start, all_limit;
  GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);

  // See if we can grow the number of inputs in "level" without
  // changing the number of "level+1" files we pick up.
  if (!c->inputs_[1].empty()) {
    std::vector<FileMetaData*> expanded0;
    current_->GetOverlappingInputs(level, &all_start, &all_limit, &expanded0);
    AddBoundaryInputs(icmp_, current_->files_[level], &expanded0);
    const int64_t inputs0_size = TotalFileSize(c->inputs_[0]);
    const int64_t inputs1_size = TotalFileSize(c->inputs_[1]);
    const int64_t expanded0_size = TotalFileSize(expanded0);
    if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size <
            ExpandedCompactionByteSizeLimit(options_)) {
      InternalKey new_start, new_limit;
      GetRange(expanded0, &new_start, &new_limit);
      std::vector<FileMetaData*> expanded1;
      current_->GetOverlappingInputs(level + 1, &new_start, &new_limit,
                                     &expanded1);
      AddBoundaryInputs(icmp_, current_->files_[level + 1], &expanded1);
      if (expanded1.size() == c->inputs_[1].size()) {
        Log(options_->info_log,
            "Expanding@%d %d+%d (%ld+%ld bytes) to %d+%d (%ld+%ld bytes)\n",
            level, int(c->inputs_[0].size()), int(c->inputs_[1].size()),
            long(inputs0_size), long(inputs1_size), int(expanded0.size()),
            int(expanded1.size()), long(expanded0_size), long(inputs1_size));
        smallest = new_start;
        largest = new_limit;
        c->inputs_[0] = expanded0;
        c->inputs_[1] = expanded1;
        GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
      }
    }
  }

  // Compute the set of grandparent files that overlap this compaction
  // (parent == level+1; grandparent == level+2)
  if (level + 2 < config::kNumLevels) {
    current_->GetOverlappingInputs(level + 2, &all_start, &all_limit,
                                   &c->grandparents_);
  }

  // Update the place where we will do the next compaction for this level.
  // We update this immediately instead of waiting for the VersionEdit
  // to be applied so that if the compaction fails, we will try a different
  // key range next time.
  compact_pointer_[level] = largest.Encode().ToString();
  c->edit_.SetCompactPointer(level, largest);
}
```

**Step1**：调用 AddBoundaryInputs 检查 inputs_[0] 中的文件 input_file 和 level 层的文件 level_file，找到**所有满足以下两个条件**的 input_file 和 level_file：

这里的 input_file 表示的是具有最大 largest_key 的 compaction file

- input_file.largest_key < level_file.smallest_key
- input_file.largest_key.user_key == level_file.smallest_key.user_key

首先查看 InternalKeyComparator 观察比较规则

第一个条件意味着 level_file 的 seqNum 相对更旧，当我们没有合并它的话，当后面查询来时找到它，然而它是一个旧值。

input_files 是若干的有交集的区间等价成一个很大的闭区间

需要明确 SST 内的数据是按 InternalKey 有序排列，我们挑选重叠 level i 层文件也是用最大和最小 InternalKey 进行比较，比较时条件也是严格大于或者小于。

返回代码中

之后调用 `GetOverlappingInputs` 得到与 level i 重叠的 level i+1 层的所有文件存放在 inputs_[1] 中，**这个函数计算时是按 user_key 计算是否重叠**

**Step2**: 其实Step1的数据就已经够合并了，但是作者 want more [狗头.jpg]，所以这里会检查是否能在不增加level+1层要合并的文件数量的同时，看看level层中还有没有文件可以进行合并。

- 计算当前 level i 和得到的 level i+1 层文件的最大最小 IntenalKey
- 然后拿这个区间再和 level i 层计算所有有重叠的文件，就得到 level i 层需要选择的合并文件，用得到的结果再和 level i + 1 计算有重叠的文件

**Step3**:

- 统计level+2层中和[all_start,all_limit]重合的文件，放到Compaction的grandparents_中，这个在IsTrivialMove中会被用于判断是否可以直接把level层文件升到level+1层（和 level i+2 有重叠，但是重叠的文件的总 size 没有超过给定阈值）。
- 更新compact_pointer_[level]的值为本次选中的level层文件的最大key(largest)，并把更新信息写到VersionEdit中。

# DoCompactionWork



# ref

1. [LevelDB源码解析(13) BackgroundCompaction SST文件合并](https://www.huliujia.com/blog/4496bd928e/)
2. [https://blog.mrcroxx.com/posts/code-reading/leveldb-made-simple/9-compaction/](https://blog.mrcroxx.com/posts/code-reading/leveldb-made-simple/9-compaction/)