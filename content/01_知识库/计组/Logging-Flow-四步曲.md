# Logging-Flow-四步曲

当用户调用 `write()` 时，底层的数据流向：

1.  **Log Writes (内存中):**
    *   `log_write()`: 进程修改 Buffer Cache 中的块。
    *   该块被标记为 Dirty，且被 Pin 住（不能驱逐回磁盘）。
2.  **Commit (写日志):**
    *   `write_log()`: 将内存中被修改的块，写入磁盘的 Log Data 区域。
    *   `write_head()`: **关键!** 将 Header Block 写入磁盘。这标志着事务提交。
3.  **Install (写原本位置):**
    *   `install_trans()`: 读取 Log Data，将其写入磁盘的 Home Location (Inode区, 数据区等)。
4.  **Clean (清理):**
    *   `write_head()`: 将 Header 中的 `n` 置为 0。表示当前日志已处理完，可以复用空间了。

---