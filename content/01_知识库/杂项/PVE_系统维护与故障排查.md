---
creation date: 2025-12-27 19:09
modification date: 2025-12-27 19:09
---
## 1. 网络异常

当系统出现异常时，按以下顺序定位问题：

### 完全失联 (Tailscale Ping 不通)

- **可能原因**：物理断电、宽带欠费、Ruijie 脚本挂死。
- **对策**：
    1. 检查物理环境（是否停电）。
    2. 若在现场，接入显示器查看屏幕输出。
    3. 尝试重插网线触发物理层重置。

### 服务不可达 (能 Ping 通但无法访问 Web)

- **Web 控制台 (8006)**：
    - 执行 `systemctl restart pveproxy` 重启 Web 服务。
- **容器服务 (如 Portainer)**：
    - 检查 NAT 规则：`iptables -t nat -L -v`。
    - 检查容器状态：`pct status 100`。

### 网络异常 (容器无法上网)

- **现象**：容器内 `apt update` 失败
- **检查清单**：
    1. **路由转发**：`cat /proc/sys/net/ipv4/ip_forward` 是否为 1？
    2. **DNS**：容器内 `/etc/resolv.conf` 是否正确？
    3. **MTU 问题**：若 HTTPS 连接重置，尝试将 MTU 降至 1400。

## 2. 数据安全

严格区分“后悔药”与“保险箱”的使用场景。

|**操作类型**|**英文名**|**存储位置**|**适用场景**|**关键动作**|
|---|---|---|---|---|
|**快照**|Snapshot|原始磁盘 (cow)|系统升级、危险实验、测试配置|实验结束后**必须删除**，防止磁盘性能下降。|
|**备份**|Backup|**Data-240G**|长期归档、寒假离校前、重要数据保存|务必确认 Storage 选对，严禁存入 `local`。|

## 3. 常用运维命令速查

```Bash
# === 网络类 ===
# 强制释放并重新获取宿主机 IP
dhclient -r enp3s0 && dhclient -v enp3s0

# 监听内网网桥流量 (抓包)
tcpdump -i vmbr1 icmp

# === 存储类 ===
# 查看挂载点与磁盘空间
lsblk
df -h

# === 认证类 ===
# 查看保活脚本实时日志
journalctl -u ruijie -f
```