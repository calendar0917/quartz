# UART-驱动详解

UART (Universal Asynchronous Receiver/Transmitter) 驱动是 Lab 中最难读懂的代码之一，尤其是**输出路径**。

## 输入 (Input) - 简单
中断来了 -> 读字符 -> 存 Buffer -> `wakeup` 读进程。

## 输出 (Output) - 困难
Shell 调用 `printf` 想打印一串字符。
1.  **放入 Buffer:** Shell 将字符放入输出 Buffer。
2.  **启动脚 (The Kick):** 
    *   如果此时 UART 是空闲的，它**不会**产生中断。
    *   驱动必须**手动**取出 Buffer 的第一个字符，写给 UART 硬件 (`uartstart`)。
    *   *这一步至关重要，就像给摩托车点火。*
3.  **后续字符 (The Flow):**
    *   当 UART 硬件把第一个字符发完后，它变得空闲，**触发“发送完成”中断**。
    *   中断处理函数 (`uartintr`) 再次调用 `uartstart`。
    *   `uartstart` 从 Buffer 取下一个字符给硬件。
    *   循环直到 Buffer 为空。

## 竞态条件
如果 Shell 正在往 Buffer 放数据，同时 UART 发完了一个字符触发中断，两者都会操作 Buffer 指针。必须全程加锁 (`uart_tx_lock`)。

---