# Usertrap-逻辑分发

这是 `trap.c` 中的 C 函数。汇编脏活干完了，这里处理逻辑。

## 预检 (Sanity Check)
*   检查 `sstatus` 寄存器的 SPP 位，确保 Trap 确实来自 User Mode。如果来自 Kernel Mode 却进了 `usertrap`，说明内核有 Bug，直接 Panic。

## 更改 stvec
*   `w_stvec((uint64)kernelvec);`
*   **战术意义:** 一旦进入内核 C 代码执行，如果此时发生中断（如时钟），我们希望它去执行 `kernelvec`（内核态中断处理），而不是 `uservec`。

## 原因分发 (scause)
1.  **Syscall (8):**
    *   检测进程是否被 Kill。
    *   **关键动作:** `p->trapframe->epc += 4;` (跳过 `ecall` 指令，否则返回后会无限递归执行 `ecall`)。
    *   开启中断 (`intr_on()`)，调用 `syscall()`。
2.  **Device Interrupt:**
    *   调用 `devintr()`。
    *   如果是时钟中断，调用 `yield()` 让出 CPU (进程调度)。
3.  **Exception (其他):**
    *   缺页、非法指令等。打印错误并杀掉进程 (`setkilled(p)`).

---