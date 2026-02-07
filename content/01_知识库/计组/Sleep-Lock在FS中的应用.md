# Sleep-Lock在FS中的应用

在文件系统中，为什么不用 Spinlock 而用 Sleep Lock？

## 核心矛盾
磁盘 I/O 是**毫秒级**的操作。
*   如果持有 Spinlock 去读磁盘，CPU 会空转几百万个周期，这是不可接受的。
*   如果持有 Spinlock 发生了中断或 Sleep，会导致死锁（见第五章笔记）。

## Sleep Lock 机制
*   `b->lock` (Buffer Lock) 和 `ip->lock` (Inode Lock) 都是 Sleep Lock。
*   **行为:** 当一个进程获取不到锁时，它会调用 `sleep()` 让出 CPU，进入睡眠状态。
*   **优势:** 允许在持有锁期间进行长时间的 I/O 操作，而不浪费 CPU 资源。

---