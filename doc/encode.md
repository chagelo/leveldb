# store key, value

使用一个标志一起存储 key，value，比如 sign|key|value

# variant encode

自然的一种想法是 keysize|key|valuesize|value，如果 key 和 value 占用的有效字节比较小，那么 size 域开销就相对多了


levelDB 的解决办法是把 size 域每个字节的第 0 位设置为标志位，1-7 位为数据域，如果为 0 表示 size 域到当前字节结束，下一个字节就是数据域。如果为 1，表示下一个字节还是 size 域，这样 size 域也是变长的了