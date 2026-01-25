---
creation date: 2025-12-11 09:30
modification date: 2025-12-11 09:30
---
# 🗂️ Sway 文件交互与 XDG 协议栈构建

## 1. 问题定义的 (Problem Definition)
Sway 是一个窗口管理器（Window Manager），而非桌面环境（Desktop Environment）。
它原生**缺乏**以下机制：
1.  **MIME 关联**：不知道 `inode/directory`（文件夹）该用什么软件打开。
2.  **DBus 通信**：不知道如何响应“在文件夹中显示”的请求。
3.  **Portal 桥接**：沙盒应用（如 Flatpak 版 Obsidian）无法穿透隔离层调用系统的文件选择器。

## 2. 架构组件 (The Stack)

### Layer 1: 文件管理器 (GUI Backend)
我们选择 **Thunar**。
*   **特性**：轻量、支持守护进程、完美兼容 Wayland。
*   **关键依赖**：`gvfs` (GNOME Virtual File System) —— **必须安装**，否则无法挂载 U 盘、无法访问垃圾桶、无法访问 SMB 网络共享。

### Layer 2: 协议路由 (MIME Types)
使用 `xdg-utils` 定义默认打开方式。告诉系统：“遇到文件夹，呼叫 Thunar”。

### Layer 3: 沙盒传送门 (XDG Portals)
Flatpak 应用处于容器中，必须通过 `xdg-desktop-portal` 请求宿主机资源。
*   `xdg-desktop-portal-gtk`: 提供文件选择对话框（File Picker）。
*   `xdg-desktop-portal-wlr`: 提供屏幕共享/录制功能（Screencast）。

## 3. 部署清单 (Implementation)

### 📦 1. 安装基础组件
```bash
# Thunar 全家桶 + Portal 实现
sudo pacman -S thunar gvfs thunar-archive-plugin file-roller \
    xdg-desktop-portal xdg-desktop-portal-gtk xdg-desktop-portal-wlr
```

### 🔗 2. 建立 MIME 关联
```bash
# 设置默认文件管理器
xdg-mime default thunar.desktop inode/directory

# 验证
xdg-mime query default inode/directory
# 应输出: thunar.desktop
```

### 🌉 3. 配置 Portal 策略 (关键)
Arch Linux 需要显式配置 Portal 后端。
编辑 `~/.config/xdg-desktop-portal/portals.conf`:
```ini
[preferred]
# 默认使用 GTK 实现 (处理文件选择、打印)
default=gtk
# 截图和录屏使用 wlr 实现 (Sway 专用)
org.freedesktop.impl.portal.ScreenCast=wlr
org.freedesktop.impl.portal.Screenshot=wlr
```

### 💉 4. 注入 Sway 环境
Portal 服务启动时必须知道当前桌面环境是 Sway。
编辑 `~/.config/sway/config`:
```sway
# 启动时更新 DBus 环境，这决定了 Portal 能否正确识别 Sway
exec dbus-update-activation-environment --systemd WAYLAND_DISPLAY XDG_CURRENT_DESKTOP=sway

# 绑定快捷键 (Mod+e 打开文件管理器)
bindsym $mod+e exec thunar
```

## 4. 验证与排错 (Verification)

### 测试手段
1.  **命令行测试**: `xdg-open .` (应弹出 Thunar)。
2.  **Flatpak 测试**: 打开 Obsidian -> 插入图片 -> 应弹出文件选择框。

### 常见故障
*   **现象**: Flatpak 点击打开文件无反应，且终端无报错。
*   **诊断**: 检查 Portal 服务状态。
    ```bash
    systemctl --user status xdg-desktop-portal
    ```
*   **修复**:
    如果服务报错或未启动，尝试重启服务或系统：
    ```bash
    systemctl --user restart xdg-desktop-portal
    ```
    检查 `XDG_CURRENT_DESKTOP` 变量是否存在：
    ```bash
    echo $XDG_CURRENT_DESKTOP
    # 必须输出: sway
    ```

## 5. 关联知识
- [[GNU Stow]]: 用于管理上述配置文件 (`portals.conf`).
- [[Flatpak]]: 理解沙盒机制与权限管理。
- [[D-Bus]]: Linux 进程间通信的基础，用于 Portal 和应用交互。