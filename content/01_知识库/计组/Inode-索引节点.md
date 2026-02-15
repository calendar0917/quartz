# Inode-索引节点

**Inode (Index Node)** 是文件系统中最核心的数据结构。
*   **文件名 vs Inode:** 文件名只是人类的助记符，Inode 才是文件的唯一标识。
*   **On-Disk Inode (`struct dinode`):** 存在磁盘上。包含：文件类型 (File/Dir/Device)、链接数 (nlink)、大小 (size)、数据块地址数组 (addrs)。
*   **In-Memory Inode (`struct inode`):** 存在内存中。是磁盘 Inode 的副本，但这多了 `ref` (引用计数) 和 `lock` (休眠锁)。

## 寻址机制 (Addressing)
xv6 的 Inode 如何找到数据？
*   `addrs[0..11]`: **直接块 (Direct Blocks)**。直接存储 Data Block 的编号。
*   `addrs[12]`: **间接块 (Indirect Block)**。指向一个 Block，那个 Block 里存了 256 个 Data Block 的编号。
*   **最大文件大小:** `(12 + 256) * 1024 bytes = 268 KB`。

---