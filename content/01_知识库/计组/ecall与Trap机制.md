# ecall与Trap机制

这是从用户态穿越到内核态的物理过程。理解这个过程是理解 Lab 4 (Traps) 的关键。

## 1. 触发点: `ecall` 指令
当用户程序调用 `write()` 时，库函数最终会执行汇编指令 `ecall` (Environment Call)。
*   **硬件行为 (Atomic):**
    1.  如果当前是 User Mode，**升级到 Supervisor Mode**。
    2.  保存当前 `pc` 到 `sepc`。
    3.  保存当前 Trap 原因到 `scause`。
    4.  **跳转** 到 `stvec` 指向的地址 (即 `uservec`)。

## 2. 上半场: `uservec` (trampoline.S)
此时，CPU 刚进内核，但页表还是用户的，寄存器还是用户的。除了 `sscratch`，内核一无所有。
1.  **交换 `sscratch` 和 `a0`:** `csrrw a0, sscratch, a0`。现在 `a0` 指向了 **Trapframe**，`sscratch` 保存了用户原本的 `a0`。
2.  **保存用户寄存器:** 把所有通用寄存器 (`ra`, `sp`, `gp`...) `sd` 到 Trapframe 里保存。
3.  **加载内核环境:**
    *   从 Trapframe 读取内核栈地址，写入 `sp`。
    *   从 Trapframe 读取内核页表地址，写入 `satp` (切换页表！)。
    *   执行 `sfence.vma` 刷新 TLB。
4.  **跳转:** 跳到 `usertrap()` (C语言函数)。

## 3. 中场: `usertrap` (trap.c)
C 代码接管控制权。
1.  检查 `scause`。如果是 `8` (Syscall)，则处理系统调用。如果是时钟中断，则可能通过 `yield()` 切换进程。
2.  更新 `sepc` (对于 Syscall，返回地址应该是 `ecall` 的下一条指令，所以 `sepc += 4`)。

## 4. 下半场: `usertrapret` & `userret`
准备回用户态。
1.  **`usertrapret`:** 关中断，更新 `stvec` 指向 `uservec`（为下一次进来做准备），把内核信息（如内核栈指针）填回 Trapframe。
2.  **`userret` (trampoline.S):** `uservec` 的逆过程。
    *   切换回用户页表 (`satp`)。
    *   从 Trapframe 恢复所有用户寄存器。
    *   交换 `sscratch` 和 `a0`。
    *   执行 `sret` 指令 -> **降级回 User Mode**，跳转回 `sepc`。

---