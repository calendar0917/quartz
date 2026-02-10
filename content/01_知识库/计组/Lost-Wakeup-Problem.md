# Lost-Wakeup-Problem

这是设计 Sleep/Wakeup 机制时必须解决的致命竞态条件。

## 场景还原
假设没有锁保护，P1 是消费者，P2 是生产者。
1.  **P1 (Read):** 检查缓冲区，发现是空的 (`buf->count == 0`)。
2.  **P1 (Decide):** 决定去睡觉。
3.  **[Context Switch]**: 切到 P2。
4.  **P2 (Write):** 写入数据，`buf->count = 1`。
5.  **P2 (Wakeup):** 调用 `wakeup`。但此时 P1 还没改成 `SLEEPING` 状态（还在 RUNNING），所以 `wakeup` 没找到人唤醒，**信号丢了**。
6.  **[Context Switch]**: 切回 P1。
7.  **P1 (Sleep):** 执行 `sleep()`，把自己改成 `SLEEPING`。
8.  **结局:** P1 永远沉睡，虽然缓冲区里有数据。这就是 **Lost Wakeup**。

## 根源
“检查条件”和“进入睡眠”这两个动作不是原子的。中间的空隙导致了唤醒信号的丢失。

---