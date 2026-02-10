# Timer-时钟中断

时钟中断 (Timer Interrupt) 是操作系统的 **"心脏起搏器"**。它是唯一一个不由用户行为触发，而是由物理时间触发的中断。

## 硬件机制 (CLINT)
RISC-V 硬件包含一个核心本地中断控制器 (CLINT)。
*   **`mtime`:** 一个不断自增的 64 位寄存器（实时时间）。
*   **`mtimecmp`:** 比较寄存器。当 `mtime >= mtimecmp` 时，触发一次时钟中断。
*   **重置:** 内核在处理完一次时钟中断后，必须重新设置 `mtimecmp = mtime + INTERVAL`，以预约下一次中断。

## 战术价值：抢占式调度 (Preemptive Scheduling)
如果没有时钟中断，操作系统将退化为 **"协作式多任务"**。
*   **场景:** 用户写了一个 `while(1);` 死循环。
*   **无时钟:** CPU 永远被占用，Shell 卡死，管理员无法 SSH，只能拔电源。
*   **有时钟:**
    1.  硬件强行打断死循环。
    2.  进入 `usertrap` -> `devintr`。
    3.  判断是 Timer Interrupt -> 调用 `yield()`。
    4.  `yield()` 调用 `sched()`，强行把当前进程踢下 CPU，换另一个进程运行。

## xv6 实现细节
*   xv6 的时钟中断通常在 **Machine Mode** (M态) 触发。
*   `timervec.S` (M态汇编) 负责接收这个硬件中断，然后通过软件生成一个 **Supervisor Software Interrupt**，通知 S态的内核进行调度处理。
*   *原因:* 这种设计允许 M 态（固件层）完全掌控物理时钟，而 S 态（内核层）只处理逻辑调度。

---