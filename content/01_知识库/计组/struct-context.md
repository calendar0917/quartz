# struct-context

`struct context` 是内核线程切换时的“快照”。

## 存储内容 (Callee-Saved Only)
它只保存 **被调用者保存寄存器 (Callee-Saved Registers)**：
*   `ra` (Return Address): **最关键！** 决定了 `swtch` 返回后去哪儿执行。
*   `sp` (Stack Pointer): **次关键！** 决定了当前的栈在哪里。
*   `s0` - `s11`: 通用寄存器。

## 对比 Trapframe
*   **Trapframe:** 保存用户态的所有寄存器（User -> Kernel）。
*   **Context:** 仅保存内核态的关键寄存器（Kernel Thread -> Scheduler）。
    *   *为什么不存 `a0`, `t0` 等？* 因为 `swtch` 是一个 C 函数调用。根据 RISC-V 约定，调用者 (`sched`) 会自动处理 Caller-Saved 寄存器，`swtch` 不需要管。
