---
title: NixOS 安装后问题修复
date: 2026-05-26
---

## 问题一：/home 挂载错误（btrfs subvol 丢失）

### 现象
登录 NixOS 后，`/home` 下面是空的，或者出现 `/home/@home/calendar/` 这样的路径。原本的 `~` 变成了 `/home/@home/calendar/`。

### 数据还在吗？

**数据一直在，从来没丢过。** 你的文件完好无损地躺在 `nvme0n1p8` 那个 btrfs 分区里。

出问题的只是"从哪里看"——就像你有一本放在三层书架第二层的书，但你看的是第一层自然看不到。把视线移到第二层，书还在原地。

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

nvme0n1p8 (顶层 — 相当于一本书的封面/目录)
├── @/           ← Arch 根目录（文件系统），subvolid=256
└── @home/       ← Arch /home 数据，    subvolid=257
    └── calendar/
        └── ...所有你的文件...

NixOS 默认挂载（错）→ 看到顶层，即 @/ 和 @home/，你的数据在 @home/calendar/ 下面
Arch / 修复后（对） → 看到 @home/ 子卷内部，直接就是 /home/calendar/
```

**修复后，原来的 /home 去哪了？** —— 它从未离开。修复只是让 NixOS "看对了地方"。你没有动任何数据文件。

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

---

## 附：NixOS 核心逻辑（与传统 Linux 的根本区别）

### 传统 Linux 怎么管理软件？

```
你 ─→ pacman -S firefox
      ↓
      pacman 从仓库下载 firefox 的 tar.zst
      ↓
      解压到 /usr/bin/firefox, /usr/lib/firefox/..., /etc/firefox/...
      ↓
      卸载时：删除这些文件（可能残留 /etc 下的配置）
```

**问题**：
- 全局 `/usr` 是所有包混在一起放的，像一个大垃圾场
- 两个包可能同时需要不同版本的依赖 → 依赖地狱
- A 包依赖 python3.11，B 包依赖 python3.12 → 只能选一个
- 卸载不干净，残留配置文件
- 很难做"后悔药"（回滚到上一版系统状态）

### NixOS 怎么管理？

NixOS 的核心思想只有一句话：**每个包的每个版本都放在自己独立、只读的目录里。**

```
/nix/store/
  ├── a3f4...-firefox-150.0/       ← Firefox 完整安装目录（只读）
  │     ├── bin/firefox
  │     ├── lib/...
  │     └── share/...
  ├── b7c2...-python-3.11.9/       ← Python 3.11（只读）
  ├── d5e1...-python-3.12.3/       ← Python 3.12（只读）← 可以共存！
  ├── f8a6...-opencode-1.0.0/      ← opencode 1.0（只读）
  └── 9b3d...-opencode-1.2.5/      ← opencode 1.2（只读）← 也可以共存！
```

每个包的路径都包含一个 **哈希值**（a3f4..., b7c2...），这个哈希是根据包的源码、依赖、编译选项算出来的。改了任何一点点，哈希就变了，新包就是**完全不同的另一个目录**。

### 对比图示

```
传统 Linux：                            NixOS：

/usr/                                  /nix/store/
├── bin/                                ├── a3f4...-firefox-150/
│   ├── bash        ← 谁的？混一起         │     ├── bin/firefox
│   ├── firefox     ← 谁的？混一起         │     └── lib/...
│   ├── python      ← 谁的？混一起         ├── b7c2...-python-3.11/
│   └── vim         ← 谁的？混一起         │     ├── bin/python
                                          │     └── lib/libpython.so
├── lib/                                 ├── d5e1...-python-3.12/
│   ├── libssl.so   ← 哪个版本？            │     ├── bin/python
│   ├── libpython.so                      │     └── lib/libpython.so
│   └── ...混在一起...                    └── f8a6...-opencode-1.0/
                                              └── bin/opencode
```

### 那 /usr/bin 里有什么？

**几乎没有东西。** NixOS 的 `/usr/bin` 几乎为空。包不装在那里。

那你怎么敲 `firefox` 运行浏览器？—— 答案在 `PATH`：

```bash
# 传统 Linux 的 PATH：
/usr/local/bin:/usr/bin:/bin

# NixOS 的 PATH（当前用户）：
/home/calendar/.nix-profile/bin:/nix/store/a3f4...-firefox-150/bin:/nix/store/b7c2...-python-3.11/bin:...

# NixOS 的 PATH（系统级，root）：
/run/current-system/sw/bin:/nix/store/xxxx-system-path-xxx/bin:...
```

每个用户有自己的 "profile"：一个符号链接集合，指向当前启用的包。切换包只是换符号链接，极其轻量。

### nixos-rebuild switch 到底干了什么

```
你编辑 configuration.nix
       ↓
nixos-rebuild switch
       ↓
┌──────────────────────────────────────┐
│ 1. 读取配置，算出需要哪些包            │
│ 2. 下载/构建缺失的包到 /nix/store/     │
│ 3. 生成一个新的 "system" 到 /nix/store │
│    （一个包含所有配置文件、systemd      │
│     unit、initrd 等的完整快照）         │
│ 4. 把 /run/current-system 的符号链接  │
│    指向新的 system                     │
│ 5. 按需重启或重载 systemd 服务         │
└──────────────────────────────────────┘
```

**关键**：每次 `switch` 都生成一个**全新的 system generation**。旧 generation 不删，留在 `/nix/store` 里。

```bash
# 查看所有 generation
ls -l /nix/var/nix/profiles/system-*-link
# 输出类似：
# system-1-link  → /nix/store/xxx...-nixos-system-...  (安装后第一天)
# system-2-link  → /nix/store/yyy...-nixos-system-...  (安装 discord)
# system-3-link  → /nix/store/zzz...-nixos-system-...  (切到 unstable)
```

### 回滚 — NixOS 的后悔药

```bash
# 回到上一次的系统状态（GRUB 菜单里也有这个选项）
sudo nixos-rebuild switch --rollback

# 或者直接选 GRUB 菜单里的旧 generation 启动
```

回滚瞬间完成，因为你只是在切换符号链接。**不需要重新下载任何东西。** 旧 generation 的所有文件都还在 `/nix/store` 里原封不动。

### /etc 去哪了？— 声明式配置

传统 Linux 里，你直接改文件：
```bash
sudo vim /etc/ssh/sshd_config    # 改 Port 22 → 2222
```

NixOS 里，你**不改文件，改配置**：
```nix
# configuration.nix
services.openssh = {
  enable = true;
  ports = [ 2222 ];
};
```

然后 `nixos-rebuild switch`。

NixOS 的 `/etc` 大部分文件是**只读的符号链接**，指向 `/nix/store` 里的生成文件。你直接 `vim /etc/ssh/sshd_config` 改不了——因为那是链接过去的不可变文件。

**好处**：配置集中在一个（或几个）`.nix` 文件里，git 一键管理。换电脑？拷走 `configuration.nix` + `hardware-configuration.nix`，`nixos-rebuild switch` 就还原出一样的系统。

### 你的系统现状（磁盘视角）

```
设备                挂载点    内容                    Nix/Arch 共享？
─────────────────────────────────────────────────────────
nvme0n1p9 (ext4)    /        NixOS 系统根            仅 NixOS
nvme0n1p8 (btrfs)   /home    用户数据                共享 ← 关键！
  其中 @home 子卷             calendar 的文档/配置/代码
  其中 @ 子卷                 Arch 的系统根           仅 Arch
nvme0n1p6 (vfat)    /boot    EFI + 内核              共享
nvme0n1p7           swap     交换空间                共享
```

NixOS 的 `/` 在独立的 ext4 分区（nvme0n1p9），Arch 的 `/` 在 btrfs 的 `@` 子卷里。两者互不干扰。

唯一共享的是 `/home`（也就是 btrfs 的 `@home` 子卷）和 `/boot`（EFI 分区）。这也是为什么从 Arch 切到 NixOS 后，你的所有个人文件、dotfile、代码都在原地——它们从来不在 `/` 分区上。

### 这个架构的代价和收益

| | 传统 Linux | NixOS |
|---|---|---|
| 安装软件 | `pacman -S xxx` 几秒 | 可能要先下载/构建，稍慢 |
| 卸载软件 | `pacman -R xxx`，可能有残留 | 去掉配置里一行 + rebuild，彻底干净 |
| 尝试最新版软件 | 危险，可能搞崩系统 | 在 `configuration.nix` 里改一行，崩了就回滚 |
| 重装系统 | 噩梦 | 拷走两个 `.nix` 文件就行 |
| 同时装两个版本 Python | 几乎不可能 | 默认就支持，各自独立目录 |
| 磁盘占用 | 小 | 大（旧 generation 堆积，需要 `nix-collect-garbage`） |
| 学习曲线 | 低 | 高（Nix 表达式语言，配置语法） |

