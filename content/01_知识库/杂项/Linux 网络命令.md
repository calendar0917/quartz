---
creation date: 2025-12-26 10:21
modification date: 2025-12-26 10:21
---
核心逻辑: IP 工具负责“路通不通”，Iptables负责“让不让过”和“怎么转弯”。

---

## 一、 基础配置与查询 (iproute2)

### 1. 接口与 IP (Layer 2 & 3)

- **查看**: `ip a` (IP地址), `ip link` (网卡状态/MAC)
    
- **修改 IP**: `ip addr add 192.168.100.1/24 dev vmbr1`
    
- **开关网卡**: `ip link set dev eth0 up/down`
    

### 2. 路由与网关 (Routing)

- **查看**: `ip r`
    
- **加默认网关**: `ip route add default via [网关IP]`
    
- **检查路由去向**: `ip route get 8.8.8.8` (排查回包路由神器)
    

### 3. 端口与监听 (Layer 4)

- **看谁在监听**: `ss -tulpn`
    
- **测远程端口**: `nc -zv [IP] [端口]`
    

---

## 二、 Iptables 核心：三表五链

> [!important]
> 
> Iptables 不是一个单一防火墙，它是一个报文处理引擎。数据包进入网卡后，会按顺序经过不同的“关卡”（链）。

### 1. 核心表 (Tables)

- **filter 表**: 负责“放行/拦截”（默认表）。
    
- **nat 表**: 负责“地址转换”（改 IP 或 端口）。
    

### 2. 核心链 (Chains) & 你的实战

| **链 (Chain)**   | **作用阶段**      | **你的实战场景**                    |
| --------------- | ------------- | ----------------------------- |
| **PREROUTING**  | 数据包刚进网卡，路由前   | **端口映射 (DNAT)**：外网访问容器        |
| **INPUT**       | 路由后，发现是给宿主机的  | **防火墙放行**：允许容器访问宿主机           |
| **FORWARD**     | 路由后，发现是转发给别人的 | **流量穿梭**：允许容器流量经过 PVE 出海      |
| **POSTROUTING** | 数据包准备离开网卡前    | **上网伪装 (SNAT)**：多个容器共享一个外网 IP |

---

## 三、 Iptables 常用“手术刀”命令

### 1. 流量放行 (Filter 表)

```Bash
# 查看规则并显示序号 (方便删除)
iptables -L -v -n --line-numbers

# 允许 vmbr1 网卡进入的所有流量 (解决容器 Ping 不通宿主机)
iptables -A INPUT -i vmbr1 -j ACCEPT

# 允许所有转发流量 (解决容器上不了网)
iptables -A FORWARD -j ACCEPT
```

### 2. 端口映射 (NAT 表 - 进入)

**场景**: 外网访问内网。

```Bash
# 语法: iptables -t nat -A PREROUTING -p [协议] --dport [外网端口] -j DNAT --to-destination [容器IP]:[容器端口]
iptables -t nat -A PREROUTING -p tcp --dport 9000 -j DNAT --to-destination 192.168.100.100:9000
```

### 3. 上网伪装 (NAT 表 - 出去)

**场景**: 内网容器共享宿主机网卡上外网。

```Bash
# 语法: iptables -t nat -A POSTROUTING -s [内网网段] -o [外网网卡名] -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o enp3s0 -j MASQUERADE
```

---

## 四、 配置文件持久化 (Debian/PVE)

> [!warning]
> 
> 以上所有 ip 和 iptables 命令都是临时的。

### 1. 永久修改网络 (`/etc/network/interfaces`)

这是 PVE 网络的“真经”，所有的 `ip` 和 `iptables` 命令都应写在 `post-up` 后面。

```Bash
auto vmbr1
iface vmbr1 inet static
    address 192.168.100.1/24
    bridge-ports none
    # 启动时执行: 开启转发
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward
    # 启动时执行: 端口映射
    post-up iptables -t nat -A PREROUTING -p tcp --dport 81 -j DNAT --to-destination 192.168.100.100:81
```

重启：`ifreload -a`
### 2. 永久修改 DNS (`/etc/resolv.conf`)

```Plaintext
nameserver 223.5.5.5
```

---

## 五、 排错全流程 (万能公式)

如果你发现 **“外网打不开容器网页”**，按此顺序排查：

1. **宿主机监听了吗？** `ss -tulpn | grep 80` (确保没有程序占用 80)
    
2. **NAT 规则在吗？** `iptables -t nat -L -v -n` (看 PREROUTING 有没有命中记录)
    
3. **容器是活的吗？** 在宿主机 `curl 192.168.100.100:80` (确保容器服务正常)
    
4. **回包路通吗？** 在容器内 `ip route` (确保默认网关指向了 192.168.100.1)
    
5. **转发开了吗？** `cat /proc/sys/net/ipv4/ip_forward` (必须为 1)
    
