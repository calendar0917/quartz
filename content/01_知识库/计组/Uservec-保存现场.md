# Uservec-保存现场

这是 `trampoline.S` 中的汇编代码，是 Trap 发生后 CPU 执行的第一段指令。此时页表仍是**用户页表**。

## 核心三步曲
1.  **获取立足点 (The Swap):**
    *   指令: `csrrw a0, sscratch, a0`
    *   **魔法:** 此时 `sscratch` 里存着 **Trapframe 的物理地址**。交换后，`a0` 变成了 Trapframe 指针，而用户原本的 `a0` 暂时存在 `sscratch` 里。
    *   *注:* 这是唯一的“无中生有”获取内核指针的方法。

2.  **全量备份 (Save All):**
    *   指令: `sd ra, 40(a0)`, `sd sp, 48(a0)` ...
    *   利用 `a0` 指向的 Trapframe，将 31 个通用寄存器全部写入内存。

3.  **世界切换 (The Switch):**
    *   **加载内核环境:** 从 Trapframe 中读取预存的 `kernel_sp` (内核栈) 到 `sp`，读取 `kernel_trap` 到 `t0`。
    *   **加载内核页表:** 从 Trapframe 读取 `kernel_satp` 到 `t1`，然后执行 `csrw satp, t1`。
    *   **刷新:** `sfence.vma`。
    *   **跳转:** `jr t0` (跳转到 `usertrap`)。

**此时状态:** CPU 运行在 S-Mode，使用内核页表，使用内核栈。

---