---
creation date: 2025-12-16 11:07
modification date: 2025-12-16 11:07
---
> [!summary] 核心价值
> **Pwndbg** 是 GDB 的一个插件，专为 **CTF Pwn** 和 **逆向工程** 设计。它彻底改变了原生 GDB 简陋的界面，提供了**自动化上下文显示（Context）**、**智能内存查看（Telescope）**以及**堆/漏洞利用辅助工具**。对于 CSAPP Bomb Lab 和后续的二进制安全学习，它是必不可少的效率工具。

---

## 1. 核心界面：上下文 (Context)

每次程序暂停（断点或单步），Pwndbg 会自动显示四个面板。学会阅读这四个面板是使用它的基础。

| 面板名称 | 颜色标识 | 功能解读 |
| :--- | :--- | :--- |
| **REGISTERS** | 💡 **高亮** | 显示 CPU 寄存器的当前值。<br>🔴 **红色**：表示该寄存器在上一条指令中**刚刚发生了变化**（重点关注）。<br>跳转地址通常会解析出函数名。 |
| **DISASM** | ⚪ 白色 | 显示**当前即将执行**的汇编指令（高亮行）及其上下文。<br>右侧会自动注释出涉及的内存值或函数参数。 |
| **STACK** | 🔵 蓝色 | 显示**栈顶（RSP）**及其下方的内存数据。<br>用于快速查看函数参数、返回地址和局部变量。 |
| **BACKTRACE** | 🟢 绿色 | 显示函数调用链（Call Stack）。<br>例如：`main -> phase_2 -> read_six_numbers`。 |

---

## 2. 常用控制指令 (Control)

比原生 GDB 更智能的流程控制。

| 指令 | 简写 | 描述 | 适用场景 |
| :--- | :--- | :--- | :--- |
| `start` | - | 自动在 `main` 函数入口断下并运行。 | 代替 `b main` + `r`，快速启动调试。 |
| `next` | `ni` | **单步步过** (Next Instruction)。遇到 `call` 不跟进。 | 快速略过标准库函数（如 `printf`）。 |
| `step` | `si` | **单步步入** (Step Instruction)。遇到 `call` 会跟进。 | 进入关键函数（如 `phase_2`）内部分析。 |
| `finish` | `fin` | 执行直到当前函数返回。 | 如果误入 `printf`，用这个跳出来。 |
| `continue` | `c` | 继续运行直到下一个断点。 | 跳过循环或不重要的代码段。 |
| `entry` | - | 运行到程序的入口点（_start）。 | 调试没有 main 函数的 stripped 二进制。 |

---

## 3. 内存神技 (Memory Inspection)

这是 Pwndbg 区别于原生 GDB 的杀手级功能。

### 🔭 Telescope (内存透视)
> [!tip] 核心指令
> **`telescope`** (简写 `tele`) 是 `x/gx` 的超级进化版。

* **功能**：打印内存内容，并**递归解析**指针。
* **语法**：`tele [地址/寄存器] [行数]`
* **示例**：
    * `tele $rsp`：查看栈上的数据（Bomb Lab 找参数必备）。
    * `tele $rdi`：查看第一个参数指向的内容（可能是字符串、链表节点）。
    * `tele 0x402460 10`：查看从该地址开始的 10 行数据（常用于查看 Switch 跳转表）。

### 🔍 Search (内存搜索)
* **功能**：在内存中搜索字符串、数值或字节序列。
* **语法**：`search [选项] "内容"`
* **示例**：
    * `search -t string "Dr. Evil"`：查找内存中是否包含该字符串。
    * `search 0xdeadbeef`：查找包含该数值的内存地址。

### 🗺️ Vmmap (内存布局)
* **功能**：显示进程的虚拟内存映射。
* **用途**：
    * 判断地址是在 **Stack**（栈）、**Heap**（堆）还是 **Text**（代码段）。
    * 检查段的权限（rwx），看哪里可写、哪里可执行。

### 🔢 Hexdump (十六进制转储)
* **功能**：以传统的十六进制+ASCII 视图查看内存。
* **语法**：`hexdump [地址] [长度]`
* **场景**：查看大块数据或数组（如 Bomb Lab Phase 6 的排序后链表）。

---

## 4. 实战 Bomb Lab 专用流 (Workflow)

针对 CSAPP Bomb Lab 的高效操作流。

### 步骤一：智能启动
不要每次手动输入答案，使用重定向。
```bash
# 在 GDB 外创建答案文件
echo "Border relations with Canada have never been better." > solution.txt
# 在 GDB 内
pwndbg> run < solution.txt
````

### 步骤二：定位 Switch 跳转表 (Phase 3)

当看到 `jmp qword ptr [rax*8 + 0x402460]` 时：

```bash
# 直接透视跳转表，列出所有可能的跳转目标
pwndbg> tele 0x402460 10
```

### 步骤三：解析链表/结构体 (Phase 6)

假设 `$rdx` 指向链表节点：

```bash
# 连续查看节点内容 (假设节点大小为 16 字节)
pwndbg> tele $rdx
# 或者是
pwndbg> x/3gx $rdx  # 查看 Value, Index, Next_Ptr
```

### 步骤四：动态修改寄存器 (Patching)

如果你卡在某个 check，想强行通过验证：

```bash
# 假设 eax 为 0 导致爆炸，强行改为 1
pwndbg> set $eax = 1
# 或者修改内存
pwndbg> set *0x402460 = 0x1234
```

-----

## 5\. 高级功能 (Advanced Pwn)

主要用于溢出攻击（Stack Overflow）和漏洞利用，Bomb Lab 可能用不到，但值得记录。

  * **`checksec`**：检查二进制文件的安全保护机制（Canary, NX, PIE, RELRO）。
  * **`cyclic [N]`**：生成 N 个字符的循环字符串，用于计算缓冲区溢出偏移量。
  * **`cyclic -l [sub_str]`**：查找崩溃时的字符串在 pattern 中的偏移量。
  * **`rop` / `ropper`**：自动搜索 ROP gadgets（用于绕过 NX 保护）。
  * **`got` / `plt`**：查看全局偏移表和过程链接表。

-----

## 6\. 配置与故障排查

配置文件路径：`~/.gdbinit`

### 常用配置命令 (在 GDB 内部)

  * **`config`**：列出所有可配置项。
  * **`theme`**：列出所有颜色和主题配置。
  * **清屏设置**（推荐开启，防止旧数据干扰）：
    ```bash
    set context-clear-screen on
    ```
  * **调整面板显示内容**：
    ```bash
    set context-sections "regs disasm stack backtrace"
    ```
