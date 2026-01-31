# PageFault-Handler

在 xv6 中，Page Fault 也是通过 `usertrap()` 进行分发的。

## 处理逻辑
1.  **识别:** 检查 `r_scause()` 是否为 13 或 15。
2.  **定位:** 读取 `r_stval()` 获取出错的虚拟地址 (VA)。
3.  **合法性检查 (Sanity Check):**
    *   VA 是否在合法范围内？(如低于 `p->sz`，高于 Stack 底)。
    *   VA 是否落在了 Guard Page（隔离页）上？
4.  **补救 (Remedy):**
    *   如果是 Lazy Allocation: `kalloc()` 分配物理页，`mappages()` 建立映射。
    *   如果是 COW: 分配新页，拷贝旧页内容，修改权限为 RW。
5.  **恢复:** 如果补救成功，直接返回 (`usertrapret`)。CPU 会重新执行刚才那条指令，这次 PTE 有效了，顺利通过。

**注意:** 如果补救失败（如内存耗尽），必须杀掉进程 (`p->killed = 1`)。

---