---
creation date: 2025-12-05 18:00
modification date: 2025-12-05 18:00
---
### 缩放
#### niri 默认配置
编辑 `~/.config/niri/...`，其中的 output - scale

但是对于 X11 旧应用，或没开启 Wayland 模式的 Electron 应用可能会变得**模糊**。这是因为 Wayland 把它们放大时用了简单的线性插值。

**对策：**

1. **强制使用 [[Wayland]] 模式**：
    - Chrome/Edge: 地址栏输 chrome://flags -> Preferred Ozone platform -> Wayland。
    - VS Code / Electron 应用: 启动参数加 --ozone-platform=wayland。
    - Obsidian: 同样加参数。
2. **如果必须用 X11 应用且不想模糊**：
    - 这很难办。Niri 目前处理 XWayland 缩放的方式就是拉伸。
    - 要么忍受模糊，要么把 scale 设为 1.0，然后去调整 **字体大小 (Font Size)** 而不是 **屏幕缩放**。
#### 调整 GTK 缩放
直接调整 GTK 的字体缩放因子：

```bash
# 1.2 表示字体放大 20%
gsettings set org.gnome.desktop.interface text-scaling-factor 1.2
```
#### flatpak 下的 obsidian
如果直接通过 kanshi 调整 scale，obsidian 在 flatpak 中通过 xwayland 兼容层运行，不支持 x11 的分数缩放，故直接 1.0 缩放，然后渲染为更大的，导致模糊。

解决方法：强制设置为 wayland

```bash
# 1. 允许访问 Wayland Socket (通常默认开启，但为了保险)
flatpak override --user --socket=wayland md.obsidian.Obsidian

# 2. 注入 Electron 标志，强制使用 Wayland 渲染
flatpak override --user --env=ELECTRON_OZONE_PLATFORM_HINT=wayland md.obsidian.Obsidian
```

可以通过 flatseal 可视化管理
### 下载
[[aria2]]:多线程，走代理

qibit 下磁力
### 无法打开软件
niri 熄屏后，发现 linuxqq、wechat、obsidian 全都打不开了……查了下资料，好像是对 electron 的兼容有问题，无解。

于是用 flatpak 重装了这些应用，除了 obsidian 都可以打开了。还是没办法，换回 kde 了

找到了解决方法：删除crash_files文件夹下的文件就好了，并将其设为只读可以避免问题复现。

`sudo rm -rf  ~/.config/QQ/crash_files/ && sudo chattr +i ...`
### 输入法问题
在 obsidian 中打字会出现漏出字母（主要是元音）的现象：

> 如果应用（Obsidian）和输入法框架（$Fcitx5$）之间的通信链路（即环境变量）被 Flatpak 沙盒切断或配置错误，应用可能只接收到**部分**或**错误**的输入信号，导致预编辑的拼音字符丢失，最终输出的汉字也就不完整了。

解法：强制指定环境变量：

```bash
flatpak override --user --env=GTK_IM_MODULE=fcitx --env=QT_IM_MODULE=fcitx --env=XMODIFIERS=@im=fcitx md.obsidian.Obsidian
```

### 代理
参看[[设置代理]]
### 桌面环境
niri 还是不太行，先弃用了。

参看[[KDE Plasma]]
### Java版本
可以用 `archlinux-java` 进行版本管理

```bash
archlinux-java status  
...
sudo archlinux-java set java-11-openjdk
```
### Burp Suite 激活
参考 https://github.com/mikhailde/burpsuite-pro-archlinux

jdk-21 是可以的。

但是缩放有问题，编辑下 zshrc：

```bash
export JAVA_TOOL_OPTIONS="-Dsun.java2d.uiScale=2.0 -Dsun.java2d.dpiaware=false"
```

- **`-Dsun.java2d.uiScale=2.0`**: 将 $UI$ 缩放强制设置为 **200%**。l
- **`-Dsun.java2d.dpiaware=false`**: 这是关键。它告诉 $Java$ **不要**尝试从操作系统（即 $KDE$）获取 $DPI$ 或缩放信息。