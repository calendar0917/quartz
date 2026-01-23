# Demand-Paging

## 概念
运行 `exec` 时，不需要把整个 ELF 文件从磁盘读入内存。
1.  只分配页表，PTE 标记为 Invalid，但在 PTE 中记录该页在磁盘上的位置（Block Number）。
2.  跳转到 Entry Point。
3.  **Page Fault:** CPU 发现代码不在内存。
4.  **Handler:**
    *   从磁盘读取对应的 4KB 数据块。
    *   `kalloc` 分配内存并写入。
    *   修改 PTE 为 Valid。
5.  **运行:** 继续执行。

## 意义
大大加快程序启动速度。特别是对于带图形界面的巨型程序，用户可能只用到了其中 10% 的功能，根本不需要加载 100% 的代码。

---