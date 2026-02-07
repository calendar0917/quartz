# Busy-Waiting-vs-Sleep

在等待某个条件成立时（例如 `uart` 发送缓冲区变空），有两种策略：

## 1. 忙等待 (Busy Waiting / Spinning)
```c
while(uart_tx_full()) 
    ;
```

## 2. 睡眠
```c
if(uart_tx_full())
    sleep(&uart_tx_buf, &uart_lock);
```

- 行为： 进程标记自己为 SLEEPING，调用 sched() 让出 CPU。
- 适用场景： 等待时间较长（毫秒级以上，如磁盘 I/O、用户输入）。
- 优点： 释放 CPU 给其他进程使用，提高吞吐量。