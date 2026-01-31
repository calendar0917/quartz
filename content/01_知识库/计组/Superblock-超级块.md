# Superblock-超级块

Superblock (通常位于磁盘的 Block 1) 是整个文件系统的**“宪法”**或**“配置表”**。它定义了文件系统的规模、布局和边界。

## 包含的元数据 (`struct superblock`)
*   **`magic`:** 魔数。内核读取它来验证“这是否是一个合法的 xv6 文件系统”。
*   **`size`:** 文件系统总的块数。
*   **`nlog`:** 日志区的块数。
*   **`ninodes`:** Inode 的总数。
*   **`nblocks`:** 数据块的总数。

## 战术地位
1.  **启动时读取:** 内核在挂载文件系统 (`fsinit`) 时，首先读取 Superblock。
2.  **计算边界:** 内核利用 Superblock 里的数据，算出 Inode 区从哪开始，Bitmaps 从哪开始。
    *   *例如:* `inodes_start = 2 + nlog`。
3.  **只读性:** 在 xv6 运行时，Superblock 通常是只读的（除非你动态调整分区大小，xv6 不支持）。

---