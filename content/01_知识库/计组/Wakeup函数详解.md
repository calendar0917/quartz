# Wakeup函数详解

`wakeup` 的逻辑相对简单，但必须遵守锁的协议。

## 逻辑
`void wakeup(void *chan)`
1.  遍历进程表 (`proc[NPROC]`)。
2.  对每个进程 `p`，必须先 **`acquire(&p->lock)`**。
    *   *原因:* 防止 P 正在 `sleep` 的过程中（还没切走），或者防止其他 CPU 同时修改 P 的状态。
3.  检查: `if (p->state == SLEEPING && p->chan == chan)`。
4.  唤醒: `p->state = RUNNABLE`。
5.  `release(&p->lock)`。

## 配合 Sleep
正是因为 `wakeup` 必须获取 `p->lock`，而 `sleep` 在释放条件锁之后、切换进程之前一直持有 `p->lock`，所以 **Wakeup 永远不可能插入到 Sleep 的“检查”和“状态修改”之间**。这就解决了 Lost Wakeup 问题。

---