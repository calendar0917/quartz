# The-Commit-Point

这是整个 Crash Recovery 机制的**奇点 (Singularity)**。原子性在这里发生。

## 定义
**“更新 Header Block 中的 `n` (计数器)”** 这个写磁盘操作，就是 Commit Point。

## 状态判定
*   **在写 Header 之前断电:** 磁盘上的 Header 里 `n=0`（或者旧值）。重启时，OS 认为日志是空的/无效的。之前写入 Log 数据区的 Block 全部作废。 -> **回滚 (Rollback)**。
*   **在写 Header 成功后断电:** 磁盘上的 Header 里 `n=5`。重启时，OS 看到日志里有 5 个块，且知道它们要去哪。 -> **重放 (Replay)**。

**本质:** 硬件保证对一个扇区（512字节）的写入是原子的。我们利用硬件的这个原子性，撬动了整个文件系统的原子性。

---