# Buffer-Cache机制

Buffer Cache (`bio.c`) 是文件系统与磁盘驱动之间的**中间件**。它的主要任务是：
1.  **缓存 (Caching):** 经常访问的块（如 Superblock, Inode）常驻内存，减少磁盘 I/O。
2.  **同步 (Synchronization):** 保证同一时刻只有一个内核线程在修改某个磁盘块。

## 数据结构
*   **双向链表 (Doubly Linked List):** 用于实现 **LRU (Least Recently Used)** 替换算法。
*   **`bcache.lock`:** 保护整个链表结构的自旋锁。
*   **`struct buf`:** 每个缓存块都有自己的休眠锁 (`sleep-lock`)。

## 读写流程 (`bread` / `bwrite`)
1.  **`bread`:** 在缓存中查找 Block。
    *   **Hit:** 移动到链表头（最近使用），返回 buf。
    *   **Miss:** 找链表尾部最老的 buf（LRU），复用它，调用磁盘驱动读取数据。
2.  **`bwrite`:** 将内存中的 buf 数据写入磁盘。

---