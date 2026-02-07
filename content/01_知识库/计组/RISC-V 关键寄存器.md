# RISC-V 关键寄存器

在 S-Mode (内核态) 下，OS 依靠一组**控制状态寄存器 (CSRs, Control and Status Registers)** 来掌控全局。普通指令（如 `add`）无法触碰它们，必须使用特权指令 `csrr` (read), `csrw` (write) 来操作。

## 1. `satp` (Supervisor Address Translation and Protection)
*   **功能:** **页表基址寄存器**。相当于 x86 的 `CR3`。
*   **内容:** 存放当前进程 **根页表 (Root Page Table)** 的物理地址。
*   **切换:** 当 OS 切换进程时，必须更新 `satp`，这会瞬间改变整个虚拟内存空间的映射关系（并刷新 TLB）。

## 2. `stvec` (Supervisor Trap Vector Base Address)
*   **功能:** **中断向量基址**。
*   **内容:** 指向 **Trap 处理入口代码** 的虚拟地址（在 xv6 中是 `trampoline.S` 里的 `uservec` 或 `kernelvec`）。
*   **作用:** 当发生 Trap（如 syscall 或时钟中断）时，CPU 会自动跳转到 `stvec` 指向的地址去执行。

## 3. `sepc` (Supervisor Exception Program Counter)
*   **功能:** **异常现场 PC**。
*   **作用:** 当 Trap 发生时，硬件会自动把当前的 `pc` (程序计数器) 保存到 `sepc`。
*   **恢复:** 当 Trap 处理完执行 `sret` 指令时，CPU 会把 `sepc` 的值拷回 `pc`，从而回到用户程序被打断的地方继续执行。

## 4. `scause` (Supervisor Cause)
*   **功能:** **异常原因**。
*   **作用:** 告诉内核发生了什么。是系统调用（值为 8）？是时钟中断（值为 1 | IsInterrupt）？还是缺页故障（值为 13/15）？内核根据这个值决定调用哪个处理函数。

## 5. `sscratch` (Supervisor Scratch)
*   **功能:** **临时寄存器**。
*   **魔法:** 在 xv6 中，它用于在 User/Kernel 切换的极短瞬间，保存 **Trapframe 的地址**。它是内核在还没建立栈之前唯一的救命稻草。

---