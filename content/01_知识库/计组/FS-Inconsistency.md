# FS-Inconsistency

## 定义
文件系统的不一致性是指磁盘上的元数据违反了文件系统的**不变量 (Invariants)**。

## 典型场景：新建文件 (Create File)
创建一个文件通常涉及三个磁盘写操作：
1.  **Alloc Inode:** 在 Inode 区域标记某个 Inode 为已使用。
2.  **Link Directory:** 在父目录的数据块中写入新文件名和 Inode 号。
3.  **Update Bitmap:** 也许还需要分配数据块。

## 崩溃后果
如果 OS 在第 1 步完成、第 2 步开始前断电：
*   **结果:** Inode 被标记为使用，但没有任何目录指向它。
*   **危害:** 这个 Inode 变成了“孤儿”，永远占用空间，且无法被删除（Resource Leak）。
*   **更糟的情况:** 目录指向了一个未分配的 Inode（Dangling Pointer），这可能导致读取垃圾数据或严重的权限绕过。

---