---
creation date: 2025-12-28 16:46
modification date: 2025-12-28 16:46
draft: true
---
### 🛡️ 项目代号：【Sentinel】（哨兵）
**定义：** 一个 Linux 下的**轻量级入侵检测探针 (Host-based IDS)** 原型。
**核心逻辑：** 监控系统进程 + 检测异常行为。
**技术栈：** Modern C++ (C++17/20) + Linux Syscalls + CMake。

---

### 📅 阶段一：破冰期 —— "C with Classes" (Day 1 - Day 3)
**目标：** 克服对 C++ 的恐惧，利用 AI 搭建工程脚手架。
**状态：** 抛弃 Java 的面向对象包袱，把 C++ 当作“带自动内存管理的 C”来用。

*   **任务 1：环境配置 (The Factory / AI 负责)**
    *   **指令：** 让 AI 帮你生成一个标准的 CMake 工程结构。
    *   **Prompt：** “我在 Arch Linux 上，要写一个 C++ 项目读取系统信息。帮我生成 `CMakeLists.txt`，要求：使用 C++17 标准，源代码在 `src`，头文件在 `include`，编译输出到 `build`。”
    *   **你的工作：** 跑通 `cmake .. && make`，看到 "Hello World"。

*   **任务 2：读取 `/proc` (The Gym / 你负责)**
    *   **背景：** 你做过 Bomb Lab，知道 Linux 下一切皆文件。进程信息都在 `/proc/[pid]` 里。
    *   **挑战：** 编写一个函数，读取 `/proc/self/status`，提取出 `Name` 和 `Pid`。
    *   **限制：** **严禁使用 C 语言的 `char*` 和 `fscanf`。** 必须强迫自己使用 C++ 的 `std::string`, `std::ifstream`, `std::getline`。
    *   **为什么？** `char*` 是万恶之源（缓冲区溢出），`std::string` 是现代生产力的代表。这是你从“汇编视角”上升到“工程视角”的第一课。

---

### 📅 阶段二：数据流转 —— 进程快照 (Day 4 - Day 7)
**目标：** 理解 Linux 进程管理，熟练使用 STL 容器。

*   **任务 1：遍历 `/proc` (The Gym)**
    *   **挑战：** 使用 `std::filesystem` (C++17 特性) 遍历 `/proc` 目录。
    *   **逻辑：** 找到所有以数字命名的文件夹（即 PID），进去读 `cmdline` 或 `stat` 文件。
    *   **关联 Bomb Lab：** 这里的每一个 PID 文件夹，背后都是一个正在运行的“炸弹”。

*   **任务 2：结构化存储 (The Factory + The Gym)**
    *   **AI 辅助：** “C++ 里怎么定义一个 `struct ProcessInfo` 包含 PID(int), Name(string), State(char)？”
    *   **核心动作：** 把读到的数据塞进 `std::vector<ProcessInfo>`。
    *   **思考题：** 你的 Bomb Lab 涉及到栈内存。现在的 `vector` 是在堆上分配的。如果 vector 扩容，内存地址会变吗？（这个问题决定了你对指针的安全性理解）。

---

### 📅 阶段三：可视化与交互 —— 赋予形体 (Week 2)
**目标：** 做出能在 Arch 终端里跑得很酷的 TUI (Text User Interface)。

*   **技术选型：** 既然是 C++ 新手，不要硬啃 ncurses 库（太古老）。我们用 **`FTXUI`** (一个现代 C++ TUI 库，Google 出品，非常适合“外骨骼”开发)。
*   **任务：**
    *   **AI 辅助：** “帮我写一个 FTXUI 的 Hello World，显示一个列表。”
    *   **你的工作：** 把上一阶段的 `vector<ProcessInfo>` 喂给 UI 组件。
    *   **效果：** 你现在拥有了一个简易版的 `htop`。

---

### 📅 阶段四：战略落地 —— 安全赋能 (Week 3+)
**目标：** 从“玩具”升级为“安全工具”。这是你简历上的杀手锏。

*   **核心功能：检测“幽灵进程” (Hidden Process Detection)**
    *   **原理：** 攻击者（Rootkit）通常会 Hook `getdents` 系统调用，让 `ls /proc` 看不到它，但它必须存在于内核的任务队列里。
    *   **实现逻辑（高阶）：**
        1.  通过常规方法（`std::filesystem` 遍历 `/proc`）获取 PID 列表 A。
        2.  （这一步是未来的扩展，先埋伏笔）通过更底层的方式（如暴力扫描 PID 1-32768 的 `kill(pid, 0)` 测试）获取存在的 PID 列表 B。
        3.  **Diff：** 如果 PID 在 B 中存在，但不在 A 中，报警：**"Found Hidden Process!"**

---

### ⚠️ “外骨骼”使用守则 (The Cyborg Protocol)

鉴于你的基础，在执行过程中必须遵守以下**三条军规**：

1.  **GDB 强制介入法：**
    *   当你用 C++ 写完第一版代码，**必须**用 GDB 调试一次。
    *   `b main` -> `run` -> `n`。
    *   看着 C++ 的 `std::string` 在内存里到底长什么样（它其实是一个包含指针、长度、容量的结构体）。**这会把你 Bomb Lab 的底层经验和 C++ 的高层抽象连接起来。**

2.  **STL 源码阅读令：**
    *   当你觉得 `vector` 很好用时，不要只当调包侠。
    *   问 AI：“C++ vector 扩容机制是怎样的？如果我是 1.5 倍扩容，内存不够了会发生什么？”
    *   这对应了你对**系统资源**的掌控欲。

3.  **不仅是 Coding，更是 Linux Admin：**
    *   当你读取 `/proc/1/environ` 发现 Permission Denied 时，不要只用 `sudo` 解决。
    *   去查 Linux Capabilities，去理解为什么普通用户不能看 init 进程的环境变量。这是安全专业的必修课。

