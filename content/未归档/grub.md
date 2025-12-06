---
creation date: 2025-12-05 17:45
modification date: 2025-12-05 17:45
---
## 原理
**GRUB (GRand Unified Bootloader)**：“大一统引导加载程序”。

其实是一个小型的操作系统，**核心任务只有两个：**

1. **把内核搬进内存**：从硬盘读取几十 MB 的内核文件 (vmlinuz) 和初始化内存盘 (initramfs) 到 RAM 中。
2. **移交权柄 (Handover)**：设置 CPU 的寄存器（指令指针 RIP），指向内核的入口地址，然后自己“自杀”（从内存中卸载），让内核接管一切。

## 整合 windows
### 软件
需要 `grub` 和 `efibootmgr` 软件包。

```bash
pacman -S grub efibootmgr os-prober
```

- `grub`: 启动管理器本身。
- `efibootmgr`: 用于将 GRUB 启动项写入 UEFI 固件的 NVRAM 中。
- `os-prober`: **关键！** 这个工具用于自动检测其他操作系统（如 Windows），并帮助 GRUB 生成相应的启动菜单项。

### 允许探测
为了让 `os-prober` 正常工作，需要编辑 GRUB 配置文件 `/etc/default/grub`，并确保以下行被取消注释或设置为 `true`：

```bash
# 使用您喜欢的编辑器，例如 nano
nano /etc/default/grub 
```

找到并确认或添加以下行：

```
GRUB_DISABLE_OS_PROBER=false
```

保存并退出。
### 挂载 win 的 efi 分区
为了让 `os-prober` 能够找到 Windows 的启动文件，需要**临时挂载** Windows 的 ESP。

```
# 假设将Windows ESP临时挂载到 /mnt/windows
mkdir /mnt/windows

# ⚠️ 替换 /dev/sdXW 为 Windows EFI 分区的实际设备名
mount /dev/sdXW /mnt/windows 
```
### 安装
GRUB 将被安装到已挂载的 Arch ESP (`/boot`) 中，并将其启动项写入 UEFI 固件。

```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Arch_GRUB --recheck
```

- `--efi-directory=/boot`: 指定 Arch 的 ESP 挂载点。
- `--bootloader-id=Arch_GRUB`: 这是在 UEFI 启动菜单中显示的名称。

**生成配置文件**

运行以下命令生成主配置文件。由于 `os-prober` 已安装且 Windows ESP 已挂载，此命令会自动检测 Windows 并为其添加一个启动菜单项。

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

执行完成后，终端会显示它找到了您的 Linux 内核和 **Windows Boot Manager**。
### 清理并重启

```bash
# 卸载、清理临时挂载点
umount /mnt/windows
rmdir /mnt/windows
```