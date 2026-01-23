# PLIC-中断控制器

PLIC (Platform Level Interrupt Controller) 是 RISC-V 架构中的中断路由器。

## 战术地位
外部设备（网卡、UART、磁盘）不能直接连到 CPU 核心，它们连接到 PLIC。PLIC 决定将中断转发给哪个 CPU 核（Hart）。

## 工作流程
1.  **Claim (认领):** 当 CPU 收到中断信号进入 `kernelvec` 后，必须读取 PLIC 的 Claim 寄存器，询问：“是谁在找我？” PLIC 返回设备号（如 UART 是 10）。
2.  **Handle (处理):** CPU 调用对应的驱动程序 (`uartintr`) 处理业务。
3.  **Complete (完成):** 处理完毕后，CPU 必须写回 PLIC 的 Claim 寄存器，以此通知 PLIC：“我处理完了，你可以发下一个了。”

## xv6 实现
*   `plicinit()`: 在内核启动时配置 PLIC，允许 UART 和 Virtio 磁盘产生中断。
*   `plicinithart()`: 每个 CPU 核启动时都要告诉 PLIC：“我准备好接收中断了”。

---