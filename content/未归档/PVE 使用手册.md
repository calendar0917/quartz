---
creation date: 2025-12-25 12:44
modification date: 2025-12-25 12:44
---
适用环境：UESTC Ruijie Campus Network

核心架构：PVE 8 + MAC 伪装 + Python 自动保活 + Tailscale 内网穿透

存储架构：120G (System/LXC) + 240G (Data/VMs)

---

## 1. 接入方式

无论是在宿舍（局域网）还是寒假在家（广域网），统一推荐使用 **Tailscale** IP 访问。

- **Web 控制台**：`https://100.x.y.z:8006`
    - _注_：浏览器提示“连接不安全”是正常的，点击“高级 -> 继续访问”。
- **SSH 终端**：`ssh root@100.x.y.z`
- **必要条件**：你的笔记本/手机必须安装 Tailscale 并登录同一账号。
---

## 2. 客户端配置

为了防止笔记本上的 **v2rayA** 与 **Tailscale** 打架，导致连不上 PVE 或上不了网，**每次重装系统或重置网络后必须检查**：
1. **Tailscale 启动参数 (防止 DNS 劫持)**：

```bash
sudo tailscale up --accept-dns=false --accept-routes
```
    
2. **v2rayA 路由规则 (防止流量回环)**：
    - 进入 v2rayA -> **RoutingA**。
    - 添加/检查规则：`ip(100.64.0.0/10) -> direct`。
    - _目的_：强制 Tailscale 流量直连，不走代理.
---

## 3. 存储使用

严格遵守存储分工，**严禁把大文件放入 local**，否则会导致 PVE 宿主崩溃。

|**存储名称 (ID)**|**物理盘**|**角色**|**✅ 允许存放**|**❌ 禁止存放**|
|---|---|---|---|---|
|**local**|120G (sda)|系统根目录|PVE日志, ISO **(仅限极小)**|**任何虚拟机磁盘**, **大ISO**|
|**local-lvm**|120G (sda)|快速存储|**LXC 容器** (Docker, Python)|Windows/Kali 这种大虚拟机|
|**Data-240G**|240G (sdb)|数据仓库|**ISO 镜像**, **VM 磁盘**, **备份**|无|

- **动作指南**：
    - 上传 ISO 时，目标存储务必选 **`Data-240G`**。
    - 创建 Windows/Kali 虚拟机时，Disks 选项里的 Storage 务必选 **`Data-240G`**

---

## 4. 网络维护

PVE 的联网全靠脚本自动维护。如果发现 PVE 离线（Tailscale ping 不通），按以下步骤排查。

### A. 检查状态 (SSH)

```Bash
# 查看保活服务是否在运行
systemctl status ruijie

# 查看实时日志 (这是看报错最快的方法)
journalctl -u ruijie -f
```

### B. 更新密码 (Password Rotate)

如果你的电信宽带密码改了，或者脚本一直报 `Login Fail`：

1. **笔记本抓包**：连校园网 -> F12 -> 登录 -> 抓取 Payload 里的加密长字符串。
2. 更新脚本：
    nano /root/auth/keep_alive.py -> 修改 ENCRYPTED_PASS。
3. 重启服务：
    systemctl restart ruijie

### C. 物理断电恢复

- **BIOS 设置**：确保已开启 **AC Recovery (Power On)**。
- **现象**：如果宿舍停电后再来电，Tailscale 没上线，可能是主板没自动开机，需要手动按电源键。

---

## 5. 备份与快照

做实验（比如跑勒索病毒样本、系统升级）前的**标准作业程序 (SOP)**：

1. **短期后悔药 (Snapshot)**： 
    - 选中虚拟机 -> `Snapshots` -> `Take Snapshot`。
    - _做完实验一定要删除快照_，否则磁盘会越来越慢。
2. **长期保险箱 (Backup)**：
    - 选中虚拟机 -> `Backup` -> `Backup now`。
    - **Storage**: 必须选 `Data-240G`。
    - _建议_：寒假离校前，把重要虚拟机都 Backup 一次。

---

## 6. 故障速查表

| **症状 (Symptom)**         | **可能原因 (Cause)**     | **解决方案 (Fix)**                                      |
| ------------------------ | -------------------- | --------------------------------------------------- |
| **Tailscale 连不上**        | PVE 没电 / 脚本挂了 / 端口被封 | 1. 检查物理电源<br>2. 检查 `ruijie` 服务状态<br>3. 尝试重插网线获取新 IP |
| **能 Ping 通但网页打不开**       | `pveproxy` 服务假死      | SSH 输入 `systemctl restart pveproxy`                 |
| **虚拟机无法上网**              | 网桥未桥接 / PVE 断网       | 1. 检查 VM 网卡是否连在 `vmbr0`<br>2. 检查 PVE 自己能不能 Ping 通百度 |
| **上传 ISO 报错 "No space"** | 选错了存储位置              | 确认你是不是选了 `local`？请选 `Data-240G`。                    |
| **Arch 笔记本部分网站打不开**      | DNS 冲突 / 路由环路        | 也就是第二章的内容：检查 Tailscale 参数和 v2rayA 规则。               |

---

## 7. 常用命令

- **重启网络服务**：`systemctl restart networking`
- **强制释放/获取 IP**：`dhclient -r enp3s0 && dhclient -v enp3s0`
- **查看磁盘挂载**：`lsblk` 或 `df -h`
- **手动运行认证脚本 (调试用)**：`python3 /root/auth/keep_alive.py`
---