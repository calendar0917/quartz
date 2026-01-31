---
creation date: 2025-12-27 18:58
modification date: 2025-12-27 18:58
---
## 认证服务架构

为了在无人值守的情况下保持 PVE 在线采用“Python 脚本 + Systemd 守护进程”的组合方案。
- **核心逻辑**：脚本模拟浏览器向校园网网关发送 HTTP POST 请求。
- **保活机制**：周期性检测网络连通性（Ping），一旦断网立即重发认证 Payload。
### 部署自动认证网络脚本

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


## 2. 维护流程：密码轮转 (Password Rotate)

当宽带密码变更或脚本报错 `Login Fail` 时，需执行以下标准操作流程：

1. **抓取凭证**：
    - 笔记本连接校园网，打开浏览器开发者工具 (F12)。
    - 登录成功后，在 Network 面板找到登录请求，复制 Payload 中的加密字符串。
2. **更新脚本**
3. **重启服务**：

```Bash
systemctl restart ruijie
```

4. **验证状态**：

```Bash
journalctl -u ruijie -f
# 观察日志是否显示 "Login Success" 或 "Online"
```
### 配置 Systemd 开机自启

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

### 穿透

[[PVE_网络工程]]：用 tailscale 进行穿透，注意 v2rayA 和 tailscale 的冲突。
## 硬件级容灾

软件层面的自动重连依赖于硬件的加电启动。

- **AC Recovery**：必须在 BIOS 中开启“通电自动开机”功能。防止因宿舍断电导致服务器关机后彻底失联。