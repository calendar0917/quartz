# archlinux 空间清理

## 一、磁盘空间分析（清理前先诊断）

```bash
# 查看整体磁盘使用情况
df -h

# 查看inode使用情况（小文件过多可能导致inode耗尽）
df -i

# 查找根目录下占用最大的10个目录
du -sh /* 2>/dev/null | sort -rh | head -n 10

# 查找大于1GB的文件
find / -type f -size +1G 2>/dev/null

# 交互式磁盘分析工具（推荐安装）
sudo pacman -S ncdu
ncdu /
```

---

## 二、Pacman 包管理器缓存清理 ⭐（最重要）

### 2.1 清理策略对比

| 命令 | 效果 | 安全性 | 推荐度 |
|------|------|--------|--------|
| `pacman -Sc` | 删除未安装包的缓存 | ⭐⭐⭐⭐⭐ | 高 |
| `pacman -Scc` | 删除所有缓存（包括已安装包） | ⭐⭐ | 谨慎 |
| `paccache -r` | 保留最近3个版本 | ⭐⭐⭐⭐⭐ | **最推荐** |
| `paccache -rk1` | 只保留1个版本 | ⭐⭐⭐⭐ | 高 |

### 2.2 具体操作

```bash
# 安装 pacman-contrib（提供 paccache 命令）
sudo pacman -S pacman-contrib

# 【推荐】保留每个包的最近3个版本，删除旧版本
sudo paccache -r

# 只保留1个版本（更激进）
sudo paccache -rk1

# 查看可清理的缓存大小（先预览）
sudo paccache -dvk2

# 删除所有未安装包的缓存
sudo pacman -Sc

# ⚠️ 彻底清空所有缓存（谨慎使用，降级时需要重新下载）
sudo pacman -Scc
```

### 2.3 缓存位置
```
/var/cache/pacman/pkg/  # Pacman 包缓存目录
```

---

## 三、孤立包（孤儿包）清理

```bash
# 查看孤立包列表（不被任何包依赖的包）
pacman -Qtdq

# 删除孤立包及其配置文件
sudo pacman -Rns $(pacman -Qtdq)

# 如果提示没有孤立包，会显示错误，可忽略
```

---

## 四、系统日志清理（journalctl）

### 4.1 查看日志占用
```bash
# 查看 journal 日志磁盘使用量
journalctl --disk-usage
```

### 4.2 清理命令

```bash
# 保留最近500MB日志
sudo journalctl --vacuum-size=500M

# 保留最近7天的日志
sudo journalctl --vacuum-time=7d

# 保留最近2周的日志
sudo journalctl --vacuum-time=2weeks

# 组合使用：保留500MB或30天，取先到者
sudo journalctl --vacuum-size=500M --vacuum-time=30d
```

### 4.3 永久限制日志大小（推荐配置）

编辑配置文件：
```bash
sudo nano /etc/systemd/journald.conf
```

添加/修改以下配置：
```ini
[Journal]
SystemMaxUse=500M        # 所有日志最多占用500MB
SystemMaxFileSize=100M   # 单个日志文件最大100MB
MaxRetentionSec=30d      # 最多保留30天
```

重启服务生效：
```bash
sudo systemctl restart systemd-journald
```

### 4.4 自动清理（cron定时任务）
```bash
# 编辑root的crontab
sudo crontab -e

# 添加每周日凌晨3点自动清理
0 3 * * 0 /usr/bin/journalctl --vacuum-size=500M
```

---

## 五、AUR 助手缓存清理

### 5.1 Yay 缓存清理
```bash
# 查看yay缓存目录
ls -lh ~/.cache/yay/

# 清理yay缓存（未安装的AUR包）
yay -Sc

# 手动清理
rm -rf ~/.cache/yay/*
```

### 5.2 Paru 缓存清理
```bash
# 清理paru缓存
paru -Sc

# 手动清理
rm -rf ~/.cache/paru/*
```

### 5.3 AUR构建残留清理
```bash
# 清理AUR构建目录（默认在~/或指定目录）
rm -rf ~/aur-builds/*  # 根据你的构建目录调整
```

---

## 六、旧内核清理 ⚠️

Arch Linux 是滚动发行版，通常只保留一个内核，但如果有多个内核：

```bash
# 查看已安装的内核
pacman -Q | grep linux

# 查看当前运行的内核
uname -r

# ⚠️ 删除旧内核（保留当前运行的）
# 假设要删除 linux-5.15.7-arch1-1
sudo pacman -Rns linux-5.15.7-arch1-1 linux-headers-5.15.7-arch1-1

# 清理未使用的内核模块
sudo rm -rf /usr/lib/modules/$(uname -r | cut -d '-' -f1)-*

# 更新initramfs和引导配置
sudo mkinitcpio -P
sudo grub-mkconfig -o /boot/grub/grub.cfg  # 如使用GRUB
```

---

## 七、用户缓存清理

```bash
# 查看用户缓存大小
du -sh ~/.cache/

# 清理用户缓存（安全，应用程序会重新生成）
rm -rf ~/.cache/*

# 选择性清理特定缓存
rm -rf ~/.cache/pip/        # Python pip缓存
rm -rf ~/.cache/npm/        # NPM缓存
rm -rf ~/.cache/mozilla/    # Firefox缓存
rm -rf ~/.cache/google-chrome/  # Chrome缓存
rm -rf ~/.cache/thumbnails/ # 缩略图缓存
```

---

## 八、临时文件清理

```bash
# 清理系统临时文件
sudo rm -rf /tmp/*

# 清理/var/tmp（比/tmp持久）
sudo rm -rf /var/tmp/*

# 清理systemd临时文件
sudo systemd-tmpfiles --clean
```

---

## 九、其他常见大文件位置清理

```bash
# 清理旧的日志文件
sudo find /var/log -name "*.log.*" -type f -delete
sudo find /var/log -name "*.gz" -type f -delete

# 清理core dump文件
sudo find / -name "core.*" -type f -delete 2>/dev/null

# 清理缩略图缓存
rm -rf ~/.thumbnails/*
rm -rf ~/.cache/thumbnails/*

# 清理Docker（如使用）
docker system prune -a

# 清理Flatpak（如使用）
flatpak uninstall --unused
```

---

## 十、自动化清理脚本（推荐）

创建清理脚本 `/usr/local/bin/arch-clean.sh`：

```bash
#!/bin/bash

echo "=== Arch Linux 空间清理脚本 ==="
echo ""

# 1. 清理孤立包
echo "[1/5] 清理孤立包..."
sudo pacman -Rns $(pacman -Qtdq) 2>/dev/null || echo "无孤立包"

# 2. 清理pacman缓存（保留3个版本）
echo "[2/5] 清理pacman缓存..."
sudo paccache -r

# 3. 清理journal日志
echo "[3/5] 清理系统日志..."
sudo journalctl --vacuum-size=500M

# 4. 清理临时文件
echo "[4/5] 清理临时文件..."
sudo systemd-tmpfiles --clean

# 5. 清理用户缓存（可选，取消注释启用）
# echo "[5/5] 清理用户缓存..."
# rm -rf ~/.cache/*

echo ""
echo "=== 清理完成 ==="
echo "当前磁盘使用情况："
df -h /
```

赋予执行权限：
```bash
sudo chmod +x /usr/local/bin/arch-clean.sh
```

---

## 十一、清理频率建议

| 清理项目 | 建议频率 | 可释放空间 |
|----------|----------|------------|
| Pacman缓存 | 每周或每月 | 500MB - 5GB |
| 孤立包 | 每月 | 50MB - 500MB |
| Journal日志 | 每周或配置自动限制 | 100MB - 2GB |
| AUR缓存 | 每月 | 100MB - 1GB |
| 用户缓存 | 按需 | 100MB - 2GB |
| 临时文件 | 重启时或每周 | 50MB - 500MB |

---

## 十二、⚠️ 安全注意事项

1. **清理前备份重要数据**
2. **不要删除 `/boot` 中的当前运行内核**
3. **谨慎使用 `pacman -Scc`**（彻底清空缓存后无法降级）
4. **不要手动删除 `/var/log/journal/` 目录**（使用 journalctl 命令）
5. **清理用户缓存前确保应用程序已关闭**
6. **使用 `paccache -dvk2` 先预览再清理**

---

## 十三、一键检查命令汇总

```bash
# 完整空间检查
echo "=== 磁盘使用 ===" && df -h
echo "=== Inode使用 ===" && df -i
echo "=== 大目录TOP10 ===" && du -sh /* 2>/dev/null | sort -rh | head -10
echo "=== Pacman缓存大小 ===" && du -sh /var/cache/pacman/pkg/
echo "=== Journal日志大小 ===" && journalctl --disk-usage
echo "=== 用户缓存大小 ===" && du -sh ~/.cache/
echo "=== 孤立包列表 ===" && pacman -Qtdq
```