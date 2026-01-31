# RISC-V 特权级

RISC-V 定义了三种主要的模式：

1.  **Machine Mode (M-Mode):**
    *   **上帝模式。** 机器上电时处于此模式。
    *   没有页表（直接物理地址）。
    *   主要用于配置硬件，然后尽快跳转到 S-Mode。
    *   xv6 在 `start.c` 中运行于 M 态。

2.  **Supervisor Mode (S-Mode):**
    *   **OS 内核模式。**
    *   开启了虚拟内存（页表）。
    *   可以处理 Trap 和中断。
    *   xv6 的 `main.c` 及其后续代码运行于 S 态。

3.  **User Mode (U-Mode):**
    *   **应用程序模式。**
    *   权限最低。

---