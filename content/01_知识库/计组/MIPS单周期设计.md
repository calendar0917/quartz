## 1. 取指部件 (Instruction Fetch)
* **核心逻辑**：`PC` $\to$ `Instruction Memory` $\to$ `Instruction`。同时 `PC = PC + 4`。
* **多路选择**：如果是 Branch/Jump 指令，PC 的更新值来源不同（`PC+4` vs. `Branch Target` vs. `Jump Target`）。

## 2. 执行部件 (Execute)
根据指令类型，ALU 的输入来源不同：
* **R-Type**：`Reg[rs]` 和 `Reg[rt]`。
* **I-Type (ALU)**：`Reg[rs]` 和 `Imm16` (需符号扩展)。
* **Load/Store**：`Reg[rs]` 和 `Imm16` (计算内存地址)。
* **Branch**：`Reg[rs]` 和 `Reg[rt]` (做减法比较)。

## 3. 控制信号 (Control Signals)
这是 CPU 的神经系统。控制单元（CU）根据 Opcode 生成信号：
* `RegDst`：写到 `rt` 还是 `rd`？
* `ALUSrc`：第二个操作数是寄存器还是立即数？
* `MemtoReg`：写回寄存器的数据来自 ALU 还是内存？
* `RegWrite`：是否写寄存器？（Branch/Store 不写）。
* `MemWrite`：是否写内存？（只有 Store 写）。
* `PCSrc`：下一条指令是 PC+4 还是分支目标？

## 4. 实战推演 (Mental Simulation)
以 `lw $1, 100($2)` 为例：
1.  **取指**：读指令，PC+4。
2.  **译码**：读 `$2` 的值。
3.  **执行**：ALU 计算 `$2 + 100`。
4.  **访存**：从内存读取地址处的数据。
5.  **写回**：将数据写入 `$1`。
* *注意*：这是关键路径最长的一条指令。

>[!注意] 注意
>单周期 CPU 的时钟周期取决于最长的那条**指令**执行的时间

## 5. 关联链接
* [[汇编语言]]：机器码如何映射到这些控制信号。