# keyvalue format

![](./img/keyvalformat.png)

（这里写错了，keysize 应该是 internalkeysize）

# LookupKey

这个类本质上包含数据 userkey 和 seqnum，提供了 3 个方法

1. memtable_key，用 internal_key_size 和 internal_key 构造一个 key，这个 key 不包含 value，注意 memtable 插入时插入的是包含 value 的，所以这个 memtable_key 始终是 memtable 里存储数据的前缀，查找时使用 `iter.Seek(memkey.data())`，它的作用是 "// Advance to the first entry with a key >= target"，如果迭代器确实找到了某个值，如果这个值的 user_key 和给定的 key 相同，然后看这个值的类型是否被删除，如果确实被删除了，那么就没有找到，否则用形参返回那个值的地址。不管 memtable 里找到的值是否被删除，只要找到了就返回 true
2. internal_key，构造 internal_key
3. user_key，构造 user_key


# InternalKey

这个类保存一个 InternalKey，它通过 user_key、seqnum、type 构造一个 string，它有成员函数能够返回 user_key

# ParsedInternalKey

这个类同样保存一个 InternalKey，不同的是它所有成员和成员函数都是公有的，它有 user_key、sequence、type 这三个对象


# InternalKeyComparator

自动义对 InternalKey 排序，方式为按 user_key 升序，其次按 seqNum 降序，再其次按 type 降序，也就是说，首先按 user_key 升序排列，其次操作越新越靠前，插入操作要比删除操作排布更前（seqNum 相同意味着 Type 一定相同）

```cpp
inline int InternalKeyComparator::Compare(const InternalKey& a,
                                          const InternalKey& b) const {
  return Compare(a.Encode(), b.Encode());
}
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  // Order by:
  //    increasing user key (according to user-supplied comparator)
  //    decreasing sequence number
  //    decreasing type (though sequence# should be enough to disambiguate)
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  if (r == 0) {
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```