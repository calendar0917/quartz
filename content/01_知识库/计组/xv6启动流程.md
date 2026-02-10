# xv6启动流程

## 启动链条
1.  **Power On:** CPU 处于 M Mode。执行 ROM 里的代码。
2.  **Bootloader:** 加载 OS Kernel 到内存 `0x80000000`。
3.  **`kernel/entry.S`:** 设置栈，跳转到 `start.c`。
4.  **`kernel/start.c` (M Mode):**
    *   设置 `mstatus` 寄存器，准备切换到 S Mode。
    *   设置 `mepc` 为 `main` 函数地址。
    *   设置页表硬件屏蔽（暂时）。
    *   配置时钟中断。
    *   执行 `mret` 指令 -> **硬件降级到 S Mode** 并跳转到 `main()`。
5.  **`kernel/main.c` (S Mode):**
    *   初始化各子系统 (内存、进程、锁、磁盘)。
    *   调用 `userinit()` 创建第一个用户进程 (`initcode.S`)。
    *   `scheduler()` 开始调度。
6.  **`init` 进程:** 执行 `/init` 程序，启动 Shell。

---