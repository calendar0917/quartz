## 1. 核心公式 (The Math)
这是本章唯一的计算重灾区。

### A. CPU 时间公式
$$CPU\ Time = \frac{Instructions \times CPI}{Clock\ Rate} = \frac{IC \times CPI}{f}$$
* **IC (Instruction Count)**：指令条数（由程序和编译器决定）。
* **CPI (Cycles Per Instruction)**：每条指令平均需要的时钟周期数（反映微架构效率）。
* **Clock Rate (f)**：主频（反映电路工艺）。

**推论**：
* **MIPS (Million Instructions Per Second)**：$MIPS = \frac{f}{CPI \times 10^6}$。
* *陷阱*：MIPS 不能跨架构比较性能！RISC 架构指令多但 CPI 低，CISC 指令少但 CPI 高，MIPS 高不代表程序跑得快。**唯一标准是执行时间（Execution Time）。**

### B. 阿姆达尔定律 (Amdahl's Law)
用于评估“局部优化对整体的提升”。

$$Speedup = \frac{1}{(1 - P) + \frac{P}{S}}$$

* **P (Percentage)**：可被改进部分占总时间的比例。
* **S (Speedup factor)**：改进部分加速了多少倍。

## 2. 深度解析 (Deep Dive)
* **边际效应递减**：如果一个部件（如乘法器）只占总运行时间的 20%，即使你把它加速到无限快，系统整体提升也无法超过 $1/(1-0.2) = 1.25$ 倍。
* **DevOps 启示**：不要盲目优化！先做 Profiling（性能分析），找到那个占用 80% 时间的瓶颈（Hot Spot），优化它才有效。

## 3. 期末生存指南 (Exam Survival)
* **计算题型 1**：给定两台机器的频率、CPI 和指令条数，问谁更快？
    * *解法*：死算 CPU Time，时间短的赢。
* **计算题型 2 (Amdahl)**：某浮点运算占 40%，硬件升级后浮点速度快了 10 倍，求整体加速比。
    * *解法*：代入公式 $1 / ((1-0.4) + 0.4/10)$。
* **概念坑**：CPI 是定值吗？错，CPI 是程序运行时的平均值，不同程序 CPI 不同。

## 4. 关联链接 (Links)
* [[Cloud Native]]：水平扩展 (Scale-out) vs 垂直扩展 (Scale-up) 的理论基础。
* [[算法复杂度]]：大 O 表示法是软件层面的衡量，CPU Time 是硬件层面的落地。