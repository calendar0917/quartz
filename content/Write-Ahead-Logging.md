# Write-Ahead-Logging

**WAL (预写式日志)** 是系统设计的黄金法则。

## 核心规则
**在将任何数据写入其最终位置（Home Location）之前，必须先将其写入日志区（Log Area）并确保落盘。**

## 逻辑推演
1.  **Log 写:** 把所有要修改的 Block（Inode, Directory, Bitmap）先写到磁盘上的一块专用区域（Log）。
2.  **Commit:** 在日志里写一个特殊的标志（Commit Record），表示“这组操作记录完毕”。
3.  **Install:** 只有 Commit 成功后，才把数据从 Log 搬运到原本该在的位置。

## 战术意义
如果在 Step 1 或 2 断电：日志不完整，重启时丢弃，文件系统保持旧状态（原子性：全不做）。
如果在 Step 3 断电：日志是完整的，重启时重做 Step 3（原子性：全做）。

---