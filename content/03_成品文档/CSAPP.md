---

creation date: 2025-12-12 22:57
modification date: 2025-12-12 22:57
---
# 程序的机器级表示
本章揭示了高级语言（C）如何被编译器翻译成机器语言（x86-64）。对于安全人员而言，这是理解 **控制流劫持 (Control Flow Hijacking)** 和 **逆向工程 (Reverse Engineering)** 的物理基础。掌握汇编不是为了写汇编，而是为了看透系统的“裸机”状态。

工具的学习：[[GDB-调试指南]] [[gcc]] [[objdump]] [[pwb]]

## 1. 基础设施 (Infrastructure)
* **生产资料（存储）：** [[寄存器与数据传输]] - 16个通用寄存器的政治经济学。
* **物流系统（寻址）：** [[寻址模式与指令]] - 立即数、寄存器与内存的交互 (`mov` vs `lea`)。
* **工作现场（栈）：** [[栈帧与过程调用]] - 函数调用的舞台，控制权转移的核心 (`call`/`ret`)。

## 2. 逻辑控制 (Control Logic)
* **状态检测：** [[条件码与比较]] - EFLAGS 寄存器 (`ZF`, `SF`, `OF`)。
* **流程跳转：** [[跳转与循环]] - 从 `if/switch` 到汇编的 `jmp` 表。

## 3. 数据组织 (Data Organization)
* **线性结构：** [[数组与指针运算]] - 步长与地址计算的本质。
* **异构结构：** [[数组与指针运算#2. 结构体与对齐 (Structs & Alignment)|结构体与对齐]] - 内存对齐 (`Alignment`) 作为一种强制规范。

## 4. 安全对抗 (Security)
* **漏洞根源：** [[缓冲区溢出攻击]] - 栈破坏与控制流劫持。
* **防御机制：** [[对抗缓冲区溢出]] - 栈金丝雀 (Canary)、ASLR、NX 位。
## 5. BombLab

[[归档/BombLab]]

通过 Lab 进行学习实践，难度还是比较大的。要学会汇编怎么读，内存的结构。

## AttackLab

[[归档/AttackLab]]

感觉比 BombLab 有意思一些，难在出思路，主要是学习基础的 CI 和 ROP，找链找得心累……但是学到了用 gdb 来解密程序，对栈帧的结构也认识的更清楚了，还有用缓冲区溢出来劫持程序，还是挺好玩的。

# 异常控制流 (Exceptional Control Flow)

**Core Argument:**
**ECF (异常控制流)** 是操作系统实现并发、I/O 和进程管理的基石。它不仅是硬件层面的“中断”，更是系统层面的“魔法”。对于安全人员，ECF 解释了 **Shellcode 如何运行**、**反弹 Shell 如何挂起**、以及 **Race Condition (竞争条件)** 漏洞的根源。

---

## 1. 底层机制 (The Mechanism)
* **硬件介入：** [[OS-异常与中断]] - 当 CPU 遇到 `Ctrl+C` 或 `div 0` 时发生了什么？(Interrupts vs Traps vs Faults)。
* **OS 介入：** [[系统调用接口]] - 用户态通往内核态的唯一合法桥梁 (`syscall`)。

## 2. 进程抽象 (The Process)
* **最伟大的谎言：** [[进程与上下文切换]] - 私有地址空间与逻辑控制流的幻觉。
* **生命周期：** [[进程控制与Fork]] - `fork` (分身), `execve` (夺舍), `exit` (死亡), `waitpid` (收尸)。

## 3. 信号机制 (Signals)
* **软件中断：** [[信号与处理机制]] - 进程间的异步通信 (`SIGINT`, `SIGKILL`, `SIGCHLD`)。
* **并发陷阱：** [[信号处理的并发安全]] - 为什么 `printf` 在 Signal Handler 里是不安全的？(Async-Signal-Safety)。

## Shell Lab

[[ShellLab]]: 挺有意思的 Lab,学习怎么用系统调用 + 内核信号来控制进程。但是对各种 API 有些陌生，对锁的理解也有点不清楚。