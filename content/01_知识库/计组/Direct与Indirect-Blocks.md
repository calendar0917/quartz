# Direct与Indirect-Blocks

这是 xv6 Inode 如何索引数据块的机制。设计目标是：**小文件访问快，大文件也能存。**

## 结构设计 (`addrs` 数组)
xv6 的 `struct dinode` 中有一个 `uint addrs[NDIRECT+1]` 数组 (12+1)。

1.  **Direct Blocks (直接块):**
    *   `addrs[0] ~ addrs[11]` (前 12 个)。
    *   **机制:** 数组里存的就是数据块的 Block Number。
    *   **容量:** `12 * 1KB = 12KB`。
    *   **优势:** 访问速度极快，读一次 Inode 就能直接定位数据。

2.  **Indirect Block (间接块):**
    *   `addrs[12]` (第 13 个)。
    *   **机制:** 这里存的不是数据，而是一个**索引块 (Index Block)** 的编号。
    *   **索引块:** 这个块里存满了 `uint` 类型的 Block Number。
    *   **计算:** 一个 Block 是 1024 字节，一个 `uint` 是 4 字节。所以一个索引块能存 `1024 / 4 = 256` 个地址。
    *   **容量:** `256 * 1KB = 256KB`。

## 总容量限制
xv6 默认的最大文件大小 = (12 + 256) * 1024 字节 ≈ **268 KB**。
*   对于现代应用，这太小了。
*   **Lab 9 (File System)** 的核心任务就是实现 **Doubly Indirect Block (二级间接索引)**，让文件支持到 `256 * 256 * 1KB = 64MB`。

---