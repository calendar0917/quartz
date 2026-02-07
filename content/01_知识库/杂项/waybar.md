---
creation date: 1970-01-01 08:00
modification date: 2025-12-05 17:54
---
Waybar 本质上是一个 **[[Wayland]] 客户端 (Client)**。它和你的浏览器、终端没有本质区别，都是向合成器（Compositor，即 Hyprland/Niri）申请一块画布，并在上面画画。

### 核心架构
是用 **C++** 编写的，利用了 **GTKmm**（GTK3 的 C++ 接口）作为 GUI 框架。

- **为什么要用 GTK？**
    - 手写 OpenGL 渲染字体、图标、阴影极其痛苦且低效。GTK 帮它处理了底层的绘图（Cairo）、字体渲染（Pango）和事件循环（GMainLoop）。
    - 这就解释了为什么 Waybar 可以用 CSS 来写样式——这是 GTK 的原生能力
### 协议
Waybar 使用了一个名为 **wlr-layer-shell (zwlr_layer_shell_v1)** 的 Wayland 扩展协议

启动时握手，连接 Wayland Socket。

然后定层，得到指定的展示位置。
### 扩展性
能够用 **tray** 模块进行通信，实现系统托盘的功能。
