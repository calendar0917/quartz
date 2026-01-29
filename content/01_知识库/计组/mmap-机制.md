# mmap-机制

`mmap` (Memory Map) 是 Unix 系统中最强大的 I/O 原语之一。它将文件内容直接映射到进程的虚拟地址空间，让读写文件像读写内存数组一样简单。

## 懒惰加载流程 (Lazy Loading Flow)
`mmap` 是 Page Fault 处理机制的集大成者。
1.  **调用 `mmap`：**
    *   内核**不读磁盘**。
    *   内核在进程的 **VMA (Virtual Memory Area)** 链表中记录一条新规则：`虚拟地址范围 [VA, VA+len) 映射到 文件 fd 的偏移 offset`。
    *   返回 VA 给用户。
2.  **用户读写 (Access)：**
    *   用户尝试读取 VA。
    *   PTE 为空 -> **触发 Page Fault**。
3.  **内核处理 (Handler)：**
    *   检查 fault address 是否在某个 VMA 范围内。
    *   **读盘：** 分配物理页，根据 VMA 记录的信息，从磁盘读取 4KB 数据到内存。
    *   **映射：** 修改 PTE 指向该物理页，设置权限 (R/W)。
4.  **写回 (Write-back)：**
    *   如果用户修改了内存 (Dirty Page)。
    *   当调用 `munmap` 或进程退出时，内核检查 Dirty 位，将修改过的数据写回磁盘文件。

## 战术优势
1.  **零拷贝 (Zero-Copy):** 避免了 `read/write` 系统调用中内核缓冲区到用户缓冲区的内存拷贝。
2.  **随机访问:** 像操作数组一样随机读写大文件，比 `lseek` 更高效。
3.  **共享内存:** 多个进程 `mmap` 同一个文件，物理内存中只需一份副本。

---