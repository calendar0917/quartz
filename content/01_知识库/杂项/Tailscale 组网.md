## 1. 方案特点
基于 **WireGuard** 协议的 Mesh VPN。
* **零配置**：无需处理 NAT 穿透、防火墙端口。
* **点对点**：设备之间尽量建立 P2P 直连，延迟低。
* **虚拟局域网**：所有设备分配一个 `100.x.y.z` 的虚拟 IP。

## 2. 安装与使用
* **一键安装**：
    ```bash
    curl -fsSL [https://tailscale.com/install.sh](https://tailscale.com/install.sh) | sh
    ```
* **启动与认证**：
    ```bash
    tailscale up
    ```
    复制终端输出的链接在浏览器登录即可。
* **查看状态**：
    ```bash
    tailscale ip  # 查看本机分配的虚拟 IP
    tailscale status # 查看网络内其他设备
    ```

## 3. 典型应用场景
* **SSH 远程管理**：不再需要暴露公网 22 端口，直接 `ssh user@100.x.y.z`。
* **Web 服务访问**：直接访问 `http://100.x.y.z:5000`。

> [!success] 体验评价
> 最“无脑”的方案。只要能联网，就能连通。非常适合个人多设备（手机、公司电脑、家里 NAS）的管理。