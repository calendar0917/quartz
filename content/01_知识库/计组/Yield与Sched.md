# Yield与Sched

当进程想要放弃 CPU 时（例如时间片到了，或等待 I/O），它会调用 `yield()`。

## Yield 流程
1.  **获取锁:** `acquire(&p->lock)`。保护进程状态。
2.  **改状态:** `p->state = RUNNABLE`。表示我还活着，只是暂时不跑。
3.  **调用 Sched:** `sched()`。

## Sched 流程 (Sanity Check)
`sched()` 是切换前的最后一道防线，它检查：
1.  我是否持有 `p->lock`？(必须持有)
2.  中断是否关闭？(必须关闭，防止死锁)
3.  我是否在运行？(状态必须不是 RUNNING)
4.  **执行切换:** `swtch(&p->context, &mycpu()->context)`。
    *   *注意:* 这一行代码执行完，CPU 就去运行调度器了。等它再次返回时，已经是甚至几秒钟以后了。

---