# Guard-Page

Guard Page（保护页）是一页**故意不映射**（PTE_V = 0）的虚拟内存，通常放置在用户栈（User Stack）的底部。

## 战术作用
它是一个**陷阱**，用于检测栈溢出（Stack Overflow）。
1.  **正常情况：** 栈指针 (`sp`) 在合法的栈空间内移动。
2.  **攻击/Bug：** 程序写了过深的递归或分配了过大的局部数组，导致 `sp` 向下越界。
3.  **触发：** `sp` 指向了 Guard Page 的地址。
4.  **拦截：** CPU 试图访问该地址 -> 查页表发现无效 (V=0) -> **触发 Page Fault**。

## 结果
内核捕获到异常，检查地址发现落在了 Guard Page 区域。内核判定这是栈溢出，直接**杀掉进程** (SIGSEGV)。
*   **如果没有它：** 溢出的数据会悄无声息地覆盖栈下方的其他数据（如 Heap 或 Code），导致数据破坏或更严重的控制流劫持 (ROP 前奏)。

## xv6 实现
在 xv6 的进程地址空间中，用户栈下方紧贴着一个未映射的 Guard Page。
*   **布局：** `[Stack Top] ... [Stack Bottom] [Guard Page (Invalid)]`
*   注意：Lab 5 的 Lazy Allocation 有时需要特殊处理 Guard Page，防止误判为 Lazy 增长的堆。

---