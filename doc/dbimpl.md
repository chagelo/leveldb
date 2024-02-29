# DBImpl 构造

- 初始化各种成员变量，如 table cache，FileLock，mem，imm，VersionSet

# NewDB

- 创建一个新的 manifest，向其中写入一个空的 Verion
- 写成功创建新的 current，写失败删除 manifest

# Put

构造一个 WriteBatch 进行写 buffer，然后调用 Write

# Write

1. 创建 Writer 写任务，然后放进队列里
2. 如果当前 Writer 不在队首，或者未执行完毕，就睡眠，否则往下
3. 调用 MakeRoomForWrite 确保写 mem 空间足够
4. 当且持有锁，可以对队列里的 Writer 任务进行合并，并记录下最后一个 Writer
5. 释放锁，写 mem

