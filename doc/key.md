# keyvalue format

![](./img/keyvalformat.png)

# LookupKey

这个类本质上包含数据 userkey 和 seqnum，提供了 3 个方法

1. memtable_key，用 internal_key_size 和 internal_key 构造一个 key，这个 key 不包含 value，注意 memtable 插入时插入的是包含 value 的，所以这个 memtable_key 始终是 memtable 里存储数据的前缀，查找时使用 `iter.Seek(memkey.data())`，它的作用是 "// Advance to the first entry with a key >= target"，如果迭代器确实找到了某个值，如果这个值的 user_key 和给定的 key 相同，然后看这个值的类型是否被删除，如果确实被删除了，那么就没有找到，否则用形参返回那个值的地址。不管 memtable 里找到的值是否被删除，只要找到了就返回 true
2. internal_key，构造 internal_key
3. user_key，构造 user_key