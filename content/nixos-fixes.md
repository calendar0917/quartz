---
title: NixOS 安装后问题修复
date: 2026-05-26
---

## 问题一：/home 挂载错误（btrfs subvol 丢失）

### 现象
登录 NixOS 后，`/home` 下面是空的，或者出现 `/home/@home/calendar/` 这样的路径。原本的 `~` 变成了 `/home/@home/calendar/`。

### 原理

你之前的 Arch Linux 使用 **btrfs 子卷**组织分区：

```
/dev/nvme0n1p8  (一块 btrfs 分区)
  ├── @         → 挂载到 /
  └── @home     → 挂载到 /home
```

Arch 的 `/etc/fstab` 里关键的一句：

```
UUID=xxx /home btrfs subvol=/@home 0 0
```

意思是：把 nvme0n1p8 分区里名叫 `@home` 的子卷挂到 `/home`。

**NixOS 图形安装器（calamares）不知道 btrfs 子卷的存在。** 你挂载 `/dev/nvme0n1p8` 到 `/mnt/home` 时，安装器只记录了"这块设备挂到 /home"，没记录 `subvol=/@home` 这个选项。

结果就是：NixOS 启动后把整个 nvme0n1p8 的 **btrfs 顶层目录** 挂到了 `/home`，而不是子卷。btrfs 顶层目录里只有 `@` 和 `@home` 两个"文件夹"（其实就是子卷的入口点），所以你的文件出现在 `/home/@home/calendar/`。

```
btrfs 分区结构（树状视图，简化）：

nvme0n1p8 (顶层)
├── @/           ← Arch 根目录（文件系统），subvolid=256
└── @home/       ← Arch /home 数据，    subvolid=257
    └── calendar/
        └── ...所有你的文件...

Arch 挂载方式：mount -o subvol=/@home → 直接看到 /home/calendar/
NixOS 挂载方式：mount 无 subvol    → 看到 /home/@home/calendar/
```

### 修复

进入 NixOS 系统，编辑 `/etc/nixos/hardware-configuration.nix`：

找到 `fileSystems."/home"` 这一段，修改为：

```nix
fileSystems."/home" = {
  device = "/dev/disk/by-uuid/6d5acaf3-799a-4b27-a4e7-cbb450c3095c";
  fsType = "btrfs";
  options = [
    "subvol=@home"
    "compress=zstd:3"
    "ssd"
    "discard=async"
    "space_cache=v2"
  ];
};
```

**关键就是加上 `options = ["subvol=@home"]`。**

然后在 NixOS 中重建：

```bash
# 如果在 NixOS 里 /home 已经挂错了，先卸载再重建
sudo umount /home
sudo mount /dev/nvme0n1p8 -o subvol=@home /home

# 验证：看看 /home/calendar 是否正常
ls /home/calendar

# 应用配置
sudo nixos-rebuild switch
```

---

## 问题二：软件版本旧

### 原理

NixOS 有两条发布线：

| 通道 | 更新频率 | 包版本 | 适合 |
|------|---------|--------|------|
| **stable (nixos-24.11)** | 每 6 个月一个大版本，中间只修 bug | 偏旧但稳定 | 服务器、不想折腾 |
| **unstable (nixos-unstable)** | 滚动更新，紧跟上游 | 最新 | 桌面用户、开发者 |

图形安装器默认装的是 **stable 通道**，所以 `opencode` 等软件是几个月前的版本。

**两种解法：**

#### 方案 A：切到 unstable（推荐桌面用户）

编辑 `/etc/nixos/configuration.nix`（或 `flake.nix`），把 channel 改成 unstable。

如果你用的是 **传统 configuration.nix 方式**：

```bash
# 添加 unstable channel
sudo nix-channel --add https://nixos.org/channels/nixos-unstable nixos
sudo nix-channel --update

# 修改 configuration.nix，不使用 stable 版本号
# 找到类似 system.stateVersion = "24.11"; 保持不变即可

# 重建
sudo nixos-rebuild switch --upgrade
```

如果你用的是 **flake 方式**（推荐长期维护）：

```nix
# flake.nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  };
  # ...
}
```

#### 方案 B：保留 stable，只给特定包用 unstable 版本

```nix
# configuration.nix
{ config, pkgs, ... }:

let
  unstable = import <nixos-unstable> { config = { allowUnfree = true; }; };
in
{
  # ...其他配置...

  environment.systemPackages = with pkgs; [
    # 大部分包用 stable
    git
    vim
    firefox

    # 少数你想要最新版的包，从 unstable 拿
    unstable.opencode
  ];
}
```

**前提**：需要先添加 unstable channel：
```bash
sudo nix-channel --add https://nixos.org/channels/nixos-unstable nixos-unstable
sudo nix-channel --update
```

---

## 验证一切正常

```bash
# 1. 确认 /home 正确挂载
ls /home/calendar          # 应该能看到你的文件
findmnt /home              # 应该显示 subvol=/@home

# 2. 确认软件版本
which opencode
opencode --version         # 或类似命令

# 3. 查看当前 channel
sudo nix-channel --list
```

---

## 建议：切换到flake管理（长期维护更清晰）

你guide里说repo路径在 `~/code/nixos/`，建议用flake管理。这样可以：
- 统一管理 stable/unstable 包
- `flake.lock` 锁定版本，可复现
- 方便多机器共享配置

如果你还没有，我可以帮你生成第一版 `flake.nix`。把你现有的 `configuration.nix` 和 `hardware-configuration.nix` 给我看就行。
