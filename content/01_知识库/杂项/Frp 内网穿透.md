## 1. 架构说明
Frp 是典型的 **C/S 架构**，包含两个二进制文件：
* **`frps` (Server)**：部署在有公网 IP 的机器上。
* **`frpc` (Client)**：部署在需要穿透的内网机器上。

> [!danger] 易错点
> **千万别搞混配置文件和程序！**
> * 公网端运行：`./frps -c frps.ini`
> * 内网端运行：`./frpc -c frpc.ini`

## 2. 关键配置 (`.ini`)

### 服务端 (frps.ini)
```ini
[common]
bind_port = 7000        # Frp 内部通讯端口
vhost_http_port = 8080  # (可选) HTTP 穿透对外端口
```

### 客户端 (frpc.ini)
```ini
[common]
server_addr = 8.137.xx.xx # 服务端公网 IP
server_port = 7000        # 对应服务端的 bind_port

[web_tcp]                 # 隧道名称，不能重复
type = tcp                # 协议类型，推荐 tcp 避免域名解析麻烦
local_ip = 127.0.0.1
local_port = 5000         # 本地服务端口
remote_port = 8001        # 暴露在公网的端口
```

## 3. 注意点
- 端口冲突：remote_port (如 8001) 是给用户访问的，不能和 bind_port (7000) 重复。

协议选择：
- HTTP/HTTPS：需要配置域名 (custom_domains)，适合建站。
- TCP：通用性最强，不依赖域名，直接 IP:Port 访问。