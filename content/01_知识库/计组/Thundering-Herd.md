# Thundering-Herd

惊群效应（Thundering Herd）是 Sleep/Wakeup 机制的副作用。

## 现象
1.  有 10 个进程都在等待同一个 `chan` (例如都在等同一个 pipe 可读)。
2.  Writer 写入了一个字节，调用 `wakeup(chan)`。
3.  `wakeup` 会把这 **10 个进程全部** 设为 RUNNABLE。
4.  这 10 个进程在多核上同时醒来。
5.  **竞争:** 它们都试图 `acquire(&pi->lock)`。
6.  **结果:** 只有 1 个进程抢到锁，读到了那个字节。剩下 9 个抢到锁后发现 `while` 条件又不满足了 (Buffer 空了)，只能再次 `sleep`。

## 代价
这就导致了大量的 **无效调度** 和 **锁争用**。
虽然逻辑是对的，但性能很差。现代 OS (如 Linux) 引入了 `wait_queue` 里的独占唤醒 (Exclusive Wakeup) 来解决这个问题（只唤醒一个）。

---