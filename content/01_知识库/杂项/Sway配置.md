---
creation date: 2025-12-11 08:48
modification date: 2025-12-11 08:48
---
## 1. 核心哲学 (Philosophy)
- **去景观化 (Anti-Spectacle)**: 摒弃 DE (GNOME/KDE) 的拟物隐喻，直接操作窗口容器。
- **键盘驱动 (Keyboard Driven)**: 所有的操作 (开启、关闭、移动、调整) 必须有快捷键映射。
- **组合性 (Composability)**: 利用 Unix 管道，将 `sway` (WM), `waybar` (Status), `fuzzel` (Launcher) 等独立工具组合。

## 2. 架构分层 (The Stack)

### Layer 0: 硬件抽象与输入
- **Display Protocol**: Wayland (更安全的隔离机制，取代 X11)。
- **Compositor**: `Sway` (基于 wlroots，i3 兼容)。
- **Input Method**: `Fcitx5` (需注入环境变量 `GTK_IM_MODULE` 等)。
- **Monitors**: `Kanshi` (守护进程，自动识别 `docked`/`undocked` 状态)。

### Layer 1: 视觉与交互 (UI/UX)
- **Bar**: `Waybar` (JSON 配置 + CSS 样式)。
- **Launcher**: `fuzzel` (应用启动与菜单选择)。
- **Terminal**: `kitty`
- **Notification**: `Mako` (轻量级通知守护)。
- **Wallpaper**: `swaybg` (支持多屏、随机轮播)。

### Layer 2: 脚本与自动化 (Automation)
- **Power Menu**: Shell 脚本调用 Wofi 选择 (Lock/Suspend/Shutdown)。
- **Screenshot**: `Grim` (截图) + `Slurp` (选区) + `wl-copy` (剪贴板)。
- **Autotiling**: Python 脚本，自动处理螺旋/分割布局。

## 3. 关键配置备忘 (Cheatsheet)

具体配置参考[[dotfiles]]
### 🎮 Sway Config (`~/.config/sway/config`)
- **重载配置**: `Mod + Shift + c` (触发 `exec_always`)
- **调整模式**: `Mod + r` (进入 Resize Mode)
- **生命周期**:
    - `exec`: 仅登录时执行一次 (Fcitx5, Mako)。
    - `exec_always`: 每次重载都执行 (Autotiling, Wpaperd, Kanshi 重启)。

### 🖥️ 显示器自动化 (`~/.config/kanshi/config`)
- **逻辑**: 全量状态匹配，而非增量补丁。
- **陷阱**: Profile 必须包含**所有**当前连接的屏幕（包括想关闭的 eDP-1）。
- **获取指纹**: `swaymsg -t get_outputs`

### 🌫️ 解决 XWayland 模糊
- **原理**: Electron 应用默认走 X11 缩放导致位图拉伸。
- **解法**: 注入 Flag `ELECTRON_OZONE_PLATFORM_HINT=wayland`。
- **工具**: `Flatseal` (可视化管理 Flatpak 环境变量)。

### ⌨️ 环境变量 (`/etc/environment`)
```bash
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
```
