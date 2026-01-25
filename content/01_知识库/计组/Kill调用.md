# Kill调用

在 xv6 中，`kill` 系统调用并不直接杀死进程（不像 Linux 的 `SIGKILL` 那么暴力）。它更像是一个**“死亡通知书”**。

## 1. 并没有立即死亡
`sys_kill(pid)` 实际上只做了两件事：
1.  找到目标进程 `p`。
2.  设置标志位: `p->killed = 1`。

## 2. 唤醒睡眠者 (The Kick)
这是本章的重点。如果目标进程正在 `sleep`（比如在等键盘输入），它可能永远不会醒来看到自己被杀了。
**Kill 的动作:**
```c
acquire(&p->lock);
p->killed = 1;
if(p->state == SLEEPING){
    // 强行唤醒！
    p->state = RUNNABLE;
}
release(&p->lock);
```
*   **逻辑:** 即使等待的条件（如键盘输入）没发生，我也要把你弄醒。

## 3. 目标进程的自我了断
进程醒来后（或者正在运行时），必须**主动检查** `p->killed`。
*   **检查点 1 (Trap):** 在 `usertrap` 中，每次系统调用或中断处理前后，都会检查 `if(p->killed) exit(-1);`。
*   **检查点 2 (Sleep Loop):** 在驱动程序的 `sleep` 循环中（如 `piperead`）：
```c
while(no_data){
if(p->killed) return -1; // 醒来后发现被杀了，放弃等待
sleep(...);
}
```

## 4. 无法杀死的进程
如果进程正在内核中执行某些**不可中断的操作**（比如在该版本的 xv6 中，等待磁盘 I/O 的 sleep 并没有检查 killed，或者持有某些自旋锁自旋中），那么 `kill` 是无效的。它必须等到进程回到用户态或进入下一个可中断的 sleep 才能生效。

