---
creation date: 2025-12-05 08:00
modification date: 2025-12-05 17:58
---
## 定义
**Wayland** 是 Linux 系统下旨在替代 X Window System (X11) 的下一代**显示服务器通信协议**。

它不是一个具体的软件，而是一套标准语言。它规定了**客户端**（Client，如浏览器、终端）如何与**合成器**（Compositor，即显示服务器）进行对话。

> [!abstract] 核心哲学
> **合成器即服务器 (The Compositor is the Server).**
> 在 Wayland 架构中，负责管理窗口和合成画面的程序（如 Hyprland, KWin, Mutter）直接掌握底层硬件控制权，不再依赖外部的 X Server 守护进程。

## 架构演变

### 旧制度: X11
这是一个**“胖客户端、胖服务端、中间人臃肿”**的模型。
`Client` <--> **X Server** <--> `Compositor` <--> `Kernel (DRM/KMS)`
*   **痛点**：渲染路径长，存在画面撕裂 (Tearing)，安全性极差（全屏监听）。

### 新制度: Wayland
这是一个“点对点直连”的模型。
`Client` <--> **Compositor (Hyprland/Niri)** <--> `Kernel (DRM/KMS)`
*   **优势**：
    1.  **直接渲染**：客户端直接在显存中绘制缓冲区，合成器只负责显示。
    2.  **强制 VSync**：从协议层面杜绝画面撕裂。
    3.  **沙盒隔离**：强制的输入/输出隔离。A 窗口无法读取 B 窗口的内容或输入，除非通过 [[XDG Desktop Portal]] 授权。

## 关键组件

| 组件 | 描述 | 举例 |
| :--- | :--- | :--- |
| **Compositor** | 也就是 Wayland Server。负责窗口管理、输入事件分发、画面合成。 | Hyprland, Sway, Niri, KWin, Mutter |
| **wlroots** | 一个模块化的库，帮助开发者快速构建 Compositor，无需从零造轮子。 | Hyprland, Sway 基于此 |
| **XWayland** | 兼容层。在 Wayland 下运行一个微型 X Server，让旧的 X11 应用能跑起来。 | 运行旧版 Steam 游戏或 Java 应用时需要 |
| **Layer Shell** | 扩展协议。允许状态栏、壁纸等程序作为特殊图层显示。 | Waybar, Hyprpaper |

## 关联知识
- 上游：[[Linux Kernel]], [[DRM-KMS]] (显卡驱动模型)
- 下游实现：[[Hyprland]], [[Niri]], [[KDE Plasma]]
- 工具链：[[Waybar]] (状态栏), [[Grim]] (截图), [[XDG Desktop Portal]] (权限管道)

---
*Ref: [Wayland Book](https://wayland-book.com/), [Arch Wiki - Wayland](https://wiki.archlinux.org/title/Wayland)*