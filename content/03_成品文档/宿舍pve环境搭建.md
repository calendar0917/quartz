---
creation date: 2025-12-24 20:58
modification date: 2025-12-24 20:58
---
这份复盘文档将作为你的《UESTC 校园网环境 PVE 部署与生存指南》**。

具体使用请参考：[[PVE 使用手册]]

## 🎯 核心目标

在电子科大（UESTC）复杂的校园网环境（Ruijie ePortal + 端口安全 + NAT）中，部署一台 Proxmox VE 服务器，实现：

1. **自动通过认证**：开机自动联网，断网自动重连。
2. **内网穿透**：寒假在家或外出时能通过内网 IP 访问。
3. **无人值守**：断电自启，无需物理干预。
## 第一阶段：环境准备与安装

### 1. 硬件连接

- **服务器 (PVE)**：网线连接墙上端口。
- **控制端 (笔记本)**：连接同一局域网（初期可能需要网线直连 PVE 进行配置）。
### 2. PVE 系统安装

- **下载**：官网下载 Proxmox VE ISO 镜像。
- **烧录**：使用 `Ventoy` 或 `Rufus` 制作启动盘。
- **安装**：
    - 插入 PVE，启动，按照引导走。
    - **IP 设置 (关键)**：安装时先随便设一个静态 IP（如 `192.168.10.2/24`），网关 `192.168.10.1`。这一步是为了安装顺利完成，后面会改。
### 3. 更换国内源 (国内源)

PVE 默认源在国外，速度极慢。安装好后，先在 PVE 终端（或通过 SSH）替换源。

**修改 `/etc/apt/sources.list` (Debian 系统源):**

```bash
sed -i 's/ftp.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
sed -i 's/security.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
```

**修改 `/etc/apt/sources.list.d/pve-no-subscription.list` (PVE 软件源):**

```bash
# 注释掉企业源，添加非订阅源
echo "deb https://mirrors.ustc.edu.cn/proxmox/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
```

**修改 CT 模板源:**

```bash
cp /usr/share/perl5/PVE/APLInfo.pm /usr/share/perl5/PVE/APLInfo.pm_back
sed -i 's|http://download.proxmox.com|https://mirrors.ustc.edu.cn/proxmox|g' /usr/share/perl5/PVE/APLInfo.pm
systemctl restart pveproxy
```

最后更新：`apt update && apt dist-upgrade -y`

---

## 第二阶段：攻克校园网x

这是最难的一步。学校交换机有端口安全策略（Port Security），且认证系统（ePortal）有加密。

### 1. 获取“通关凭证” (在笔记本上操作)

我们需要把 PVE 伪装成你的笔记本。
1. **连网**：笔记本插网线，正常登录校园网。
2. **记录 MAC**：终端输入 `ip link`，记录有线网卡 MAC (例如 `40:....`)。
3. **抓取加密密码 (Replay Attack 准备)**：
    - 按 `F12` 打开浏览器开发者工具 -> `Network` (网络)。
    - 勾选 `Preserve log` (保留日志)。
    - 点击登录。
    - 找到 `InterFace.do?method=login` 的请求。
    - 在 `Payload` (载荷) 中，复制超长的 **加密后 `password`** 字符串。
    - _注意：不需要后缀（如 @telecom），直接取 Payload 里的 userId。_
### 2. PVE 网络伪装 (MAC Spoofing)

SSH 进入 PVE，修改 `/etc/network/interfaces`。

```config
auto lo
iface lo inet loopback

iface enp3s0 inet manual

# 创建网桥（PVE 标准配置），但核心是在这里伪造 MAC
auto vmbr0
iface vmbr0 inet dhcp
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
    # 【关键】伪造 MAC 地址，骗过交换机
    hwaddress ether 40:...
```

_修改完后重启 PVE (`reboot`)，确保能获取到 `10.x.x.x` 或 `100.x.x.x` 的校园网 IP。_

### 3. 部署自动认证“看门狗”脚本

创建脚本 `/root/auth/keep_alive.py`。这是一个集成了**重放攻击**和**断网检测**的脚本。

```python
import requests
import time
import subprocess

# === 配置区 ===
USER = "177xxxxxxx" # 你的账号
# 粘贴抓包到的超长加密字符串
ENCRYPTED_PASS = "2384c6e64e96c925e38b3b1cb8aa902d420fa1c1f..." 
# =============

def check_internet():
    """Ping 检测网络"""
    try:
        subprocess.check_call(["ping", "-c", "1", "-W", "2", "223.5.5.5"], 
                            stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        return True
    except:
        return False

def get_qs():
    """动态获取 QueryString (IP/MAC信息)"""
    try:
        r = requests.get("http://1.1.1.1", allow_redirects=False, timeout=3)
        if "Location" in r.headers:
            return r.headers["Location"].split("?", 1)[1]
    except: pass
    return ""

def login():
    qs = get_qs()
    url = "http://172.25.249.64/eportal/InterFace.do?method=login"
    payload = {
        "userId": USER,
        "password": ENCRYPTED_PASS,
        "service": "",
        "queryString": qs,
        "passwordEncrypt": "true" # 告诉服务器这是密文
    }
    headers = {"User-Agent": "Mozilla/5.0"}
    try:
        print(f"[{time.strftime('%T')}] Trying login...")
        requests.post(url, data=payload, headers=headers, timeout=5)
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    print(">>> Watchdog started...")
    while True:
        if not check_internet():
            login()
            time.sleep(5)
            if check_internet():
                print(f"[{time.strftime('%T')}] Re-connected!")
        time.sleep(60) # 每分钟检查一次
```

---

## 第三阶段：持久化与远程访问

### 1. 配置 Systemd 开机自启

确保 PVE 重启后脚本自动运行。创建 `/etc/systemd/system/ruijie.service`：

```Ini
[Unit]
Description=Ruijie Auto Login Watchdog
After=network.target

[Service]
ExecStart=/usr/bin/python3 /root/auth/keep_alive.py
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```


```Bash
systemctl daemon-reload
systemctl enable ruijie
systemctl start ruijie
```

### 2. 安装 Tailscale (内网穿透)

为了在寒假或宿舍外访问 PVE，使用 Tailscale 打通隧道。

**在 PVE 端：**

```Bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
# 复制屏幕上的链接，用账号登录绑定
```

**在 笔记本/手机 端：**

- 安装 Tailscale 客户端，登录同一账号。    
- 使用分配的 `100.x.y.z` IP 即可随时 SSH 或访问 HTTPS 面板。
### 3. 解决 Arch Linux 笔记本冲突 (重要)

如果你的主力机是 Arch Linux 且跑了 v2raya，会和 Tailscale 冲突。

修正命令：

```Bash
sudo tailscale up --accept-dns=false --accept-routes
```

并在 v2raya 设置中，将 `100.64.0.0/10` 加入路由白名单（Bypass）。(貌似非必要)

---

## 第四阶段：物理防御与维护

### 1. BIOS 设置 (必做)

- **断电恢复 (AC Recovery)**：进入 BIOS 电源管理，设置为 `Power On`。防止宿舍断电后服务器变砖。
- **合盖不休眠**：如果是笔记本做服务器，确保 `/etc/systemd/logind.conf` 中 `HandleLidSwitch=ignore`。

### 2. 检查手段 (运维)

- **看脚本活没活**

```Bash
systemctl status ruijie
journalctl -u ruijie -f  # 实时看日志
```
 
- **看内网穿透状态**：

```Bash
tailscale status
```
### 3. 应急预案

- **密码过期/失效**：如果寒假期间连不上，可能是 Replay 的密文过期。
    - _解法_：只能回宿舍，用笔记本重新抓包，SSH 进去更新 `keep_alive.py` 里的字符串。
- **彻底断联**：
    - _解法_：Tailscale 有时候需要 PVE 重启才能恢复。如果 BIOS 设置了通电自启，可以让室友帮忙拔插一下电源。
---

## 📋 总结你的技术栈

通过这次部署，你实际上掌握了以下技能树：

- **Linux 系统管理**：Systemd 服务、包管理、网络配置。
    
- **网络攻防**：MAC 地址欺骗、中间人/重放攻击原理、DHCP/ARP 机制。
    
- **网络架构**：NAT 穿透 (Overlay Network)、路由表管理、透明代理共存。
    
- **自动化运维**：Python 脚本编写、看门狗逻辑。