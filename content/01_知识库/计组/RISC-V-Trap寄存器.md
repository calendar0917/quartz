# RISC-V-Trap寄存器

在处理 Trap 时，通用寄存器 (General Purpose Registers) 是不可信且不可用的（存着用户数据）。内核必须依赖 CSR (Control and Status Registers) 来完成“盲操”。

## 关键 CSR 解析

| 寄存器 | 全称 | 作用 | xv6 中的具体用法 |
| :--- | :--- | :--- | :--- |
| **stvec** | Supervisor Trap Vector Base | **入口地址**。Trap 发生时 PC 强制指向这里。 | 在用户态时指向 `trampoline` 的 `uservec`；在内核态时指向 `kernelvec`。 |
| **sepc** | Supervisor Exception PC | **断点记录**。保存 Trap 发生时的 PC。 | 用于 `sret` 时恢复执行。如果是 Syscall，内核需手动 `sepc += 4`。 |
| **scause** | Supervisor Cause | **案由**。记录 Trap 的类型。 | 8=Syscall, 13/15=PageFault, 1=Timer Interrupt。 |
| **sscratch** | Supervisor Scratch | **暗箱**。内核与用户交换数据的唯一抓手。 | **至关重要**。在 User Mode 下，它保存 **Trapframe 的物理地址**。 |
| **sstatus** | Supervisor Status | **状态位**。控制全局中断开启/关闭 (SIE) 及之前的模式 (SPP)。 | 用于记录进 Trap 之前是 User 还是 Kernel 模式。 |

## 战术意义
这些寄存器是硬件留给 OS 的“后门”。Trap 发生的那一微秒，软件无法干预，全靠硬件自动读写这些寄存器来保存最原始的现场。

---