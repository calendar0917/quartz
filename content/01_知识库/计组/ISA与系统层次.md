## 1. 本质 (First Principles)
* **核心概念：ISA (Instruction Set Architecture)**
    * **定义**：软件与硬件之间的**界面 (Interface)** / **契约 (Contract)**。
    * **包含内容**：指令格式、寻址方式、寄存器定义、数据类型（大端/小端）。
    * **关键区分**：
        * **ISA (架构)**：设计蓝图 (e.g., x86-64, ARMv8, RISC-V)。
        * **Microarchitecture (微架构/组成)**：具体实现 (e.g., Intel Raptor Lake, AMD Zen 4)。
    * *比喻*：ISA 是 TCP/IP 协议标准，微架构是华为或思科的具体路由器实现。只要符合协议，就能通信。

## 2. 层次结构 (Hierarchy)
从下到上：
1.  **硬件层**：晶体管 $\rightarrow$ 门电路 $\rightarrow$ 功能部件。
2.  **ISA 层**：软硬件分界线（汇编程序员/编译器作者关注的层级）。
3.  **系统软件层**：OS、编译器（负责将高级抽象翻译为 ISA 指令）。
4.  **应用层**：解决具体问题。

## 3. 直观探针 (Concept Probe)
* **为什么要有层次？**
    * **屏蔽复杂性**：你在写 Go 代码处理 HTTP 请求时，不需要知道底层寄存器是 32 位还是 64 位，这由编译器和 OS 屏蔽。
    * **抽象 (Abstraction)**：计算机科学最核心的方法论。

## 4. 关联链接 (Links)
* [[操作系统]]：OS 是直接操作硬件的第一层软件，它依赖 ISA 提供的特权指令（Privileged Instructions）来管理资源。
* [[逆向工程]]：逆向就是从机器码/汇编（ISA 层）推导高级逻辑的过程。

## 5. 期末生存指南 (Exam Survival)
* **填空/选择**：软件和硬件的交界面是什么？答案：**ISA**。
* **翻译链**：高级语言 $\xrightarrow{编译}$ 汇编语言 $\xrightarrow{汇编}$ 机器语言。