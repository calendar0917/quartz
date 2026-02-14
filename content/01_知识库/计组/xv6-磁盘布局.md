# xv6-磁盘布局

xv6 将磁盘看作一个巨大的 `block` 数组（通常每个 block 1024 字节）。它被静态划分为几个区域：

## 布局图谱 (Layout)
`[ Boot | Super | Log | Inodes | Bitmaps | Data Blocks ... ]`

1.  **Boot Block (Block 0):** 包含启动加载器代码。OS 启动时读取这里。
2.  **Super Block (Block 1):** 记录文件系统的元数据（大小、Inode 数量等）。
3.  **Log (Block 2...):** 日志区。用于 Crash Recovery（这是下一章的重点）。
4.  **Inodes:** 索引节点区。所有文件的元数据都紧凑地排布在这里。
5.  **Bitmaps:** 位图区。用 0/1 记录哪些 Data Block 是空闲的。
6.  **Data Blocks:** 数据区。占据了磁盘的绝大部分，存放真正的文件内容。

## 战术意义
这种布局在**格式化 (mkfs)** 时就已经确定，运行时不可更改区域大小。这是一种简单的设计，现代 FS (如 ext4, btrfs) 会更灵活。

---