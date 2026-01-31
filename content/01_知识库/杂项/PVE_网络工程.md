---
creation date: 2025-12-27 18:40
modification date: 2025-12-27 18:40
---
需要先完成 [[PVE_校园网自动化认证]]

参考 [[Linux 网络命令]]

## 1. 理论基础：OSI 模型与网络边界

网络配置的本质是处理 OSI 模型中不同层级的数据封装与转发。

| **层级**       | **关键协议/概念** | **PVE 实战对应物**                        | **典型问题**                   |
| ------------ | ----------- | ------------------------------------ | -------------------------- |
| **L7 应用层**   | HTTP, DNS   | **Nginx Proxy Manager**, 域名 (`.lab`) | 网页打不开，SSL 证书报错             |
| **L4 传输层**   | TCP/UDP, 端口 | **Portainer (9000)**, **NPM (81)**   | Connection refused (端口未放行) |
| **L3 网络层**   | IP, NAT     | **192.168.100.1**, `iptables`        | Network unreachable (路由不通) |
| **L2 数据链路层** | MAC 地址      | **vmbr0**, 虚拟网卡                      | 认证失败 (MAC 地址非法)            |

### 为什么桥接 (Bridge) 模式会失败？

在校园网环境中，交换机开启了 **Port Security** 或 **802.1x** 认证，只记录宿主机物理网卡的 MAC 地址。
- **冲突点**：桥接模式下，容器会生成一个新的虚拟 MAC 请求 IP。
- **结果**：上层交换机检测到未授权的 MAC 地址，直接丢弃数据包。

## 2. 解决方案：NAT (网络地址转换)

将 PVE 宿主机配置为一台“软路由器”，对内管理虚拟局域网，对外伪装流量。

### 核心机制

1. **SNAT (源地址伪装)**：容器访问互联网时，PVE 将数据包的“源 IP”修改为 PVE 的物理 IP，欺骗校园网交换机。
2. **DNAT (端口转发)**：互联网访问容器服务时，PVE 根据端口号将请求“搬运”进内网容器。

### 标准配置模板 (/etc/network/interfaces)

此配置由 Ansible 管理，包含 NAT 转发与端口映射规则。

```Bash
auto lo
iface lo inet loopback

auto enp3s0
iface enp3s0 inet dhcp
    # 必须从学校系统注册的 MAC 复制过来，防止认证失败
    hwaddress ether 40:c2:ba:59:67:d1

# BEGIN ANSIBLE MANAGED BLOCK
auto vmbr1
iface vmbr1 inet static
    address 192.168.50.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    # NAT 转发规则
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up iptables -t nat -A POSTROUTING -s '192.168.50.0/24' -o enp3s0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s '192.168.50.0/24' -o enp3s0 -j MASQUERADE

    # 将访问宿主机 enp3s0 的 80/443/81 端口转发给 Debian 虚拟机 (192.168.50.2)
    post-up iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.50.2:80
    post-up iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination 192.168.50.2:443
    post-up iptables -t nat -A PREROUTING -p tcp --dport 81 -j DNAT --to-destination 192.168.50.2:81
    
    # 删除规则（对应 post-up）
    post-down iptables -t nat -D PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.50.2:80
    post-down iptables -t nat -D PREROUTING -p tcp --dport 443 -j DNAT --to-destination 192.168.50.2:443
    post-down iptables -t nat -D PREROUTING -p tcp --dport 81 -j DNAT --to-destination 192.168.50.2:81
# END ANSIBLE MANAGED BLOCK
```

## 3. 远程接入与客户端冲突

使用 **Tailscale** 进行内网穿透时，需注意与本地代理软件的冲突。
- **客户端 (笔记本) 配置 SOP**：

 Tailscale 参数： 需要防止 DNS 被污染及路由表混乱。

```bash
sudo tailscale up --accept-dns=false --accept-routes=false --netfilter-mode=off
```

v2rayA 路由规则：

必须添加规则 ip(100.64.0.0/10) -> direct，强制 Tailscale 流量直连，防止流量回环导致无法连接 PVE。

## 4. SSH 密钥同步

```config
# ~/.ssh/config

# 全局配置：保持长连接，应对 Tailscale 网络波动
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h:%p
    ControlPersist 10m

# 中转机：PVE 主机 (Tailscale IP)
Host pve
    HostName 100.104.120.97
    User root
    # 如果 PVE 修改了 SSH 端口，请添加 Port 字段

# 目标机：Debian VM (PVE 内网 IP)
Host debian-vm
    HostName 192.168.50.2
    User your_username
    # 核心：通过 pve 自动跳转
    ProxyJump pve
```

```bash
# 生成密钥对
ssh-keygen -t ed25519
ssh-copy-id pve
ssh-copy-id debian-vm
```

## 5. 文件同步

`sshfs debian-vm:/home/youruser ~/mnt/debian_vm` 挂载

Rsync:

```bash
# -a: 归档模式 (保留权限、时间戳)
# -v: 显示详情
# -z: 压缩传输 (对 Tailscale 网络很有用)
# -P: 显示进度并允许断点续传
rsync -avzP ~/my_project/ debian-vm:~/my_project/
```