# FileLock

# Logger

# RandomAccessFile

# SequentialFile

# Slice

实际上就是 `std::string`，只不过多实现了 `remove_prefix` 和 `starts_with` 函数

# WritableFile 

writablefile 是对文件操作的抽象，是对不同操作系统的文件写操作的抽象。levelDB 包含 posix 和 windows 的实现