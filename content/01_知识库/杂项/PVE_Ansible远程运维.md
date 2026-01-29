---
creation date: 2025-12-27 19:08
modification date: 2025-12-27 19:08
---
## 1. 连接拓扑与 Inventory 策略

本环境采用典型的“堡垒机”架构。由于 Docker 所在的 Debian 虚拟机 (`192.168.50.2`) 位于 PVE 的内网 NAT 网段，无法直接从外部访问，因此必须将 PVE 宿主机作为跳板。

### 魔法参数：ProxyCommand

在 `inventory.ini` 中，我们利用 SSH 的 `ProxyCommand` 实现透明跳转。Ansible 会先连接 PVE (Tailscale IP)，再从 PVE 通过内网 SSH 到达虚拟机 111。

```TOML
[pve]
# PVE 宿主机 (Tailscale 入口)
pve_host ansible_host=100.104.120.97 ansible_user=root

[docker_vm]
# 虚拟机内网 IP (不可直连)
debian_docker ansible_host=192.168.50.2 ansible_user=root

[docker_vm:vars]
# 核心配置：定义 SSH 跳板逻辑
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -q root@100.104.120.97"'
```

## 2. PVE 宿主机配置 (`pve_host` Role)

此角色负责构建底层基础设施，核心任务是网络通过性与校园网认证保活。

### 2.1 NAT 网络构建 (Infrastructure as Code)

不再手动编辑 `/etc/network/interfaces`，而是通过 Ansible 的 `blockinfile` 模块进行幂等管理。

- **网段定义**：`192.168.50.1/24` 2。
- **SNAT (伪装)**：开启 `ip_forward` 并设置 `MASQUERADE` 规则，使内网容器可上网 3.
- **DNAT (端口映射)**：将宿主机的 80/443/81 端口流量精确转发至 `192.168.50.2` 4。

### 2.2 认证保活自动化

将 Python 脚本模板化 (`keep_alive.py.j2`)，实现敏感信息的解耦注入。

- **变量注入**：`ruijie_user` 和 `ruijie_encrypted_pass` 从 `group_vars/pve.yml` 读取并填充至脚本中。
- **服务托管**：自动部署 `ruijie.service` 并确保开机自启，实现无人值守的校园网认证。

## 3. 虚拟机初始化 (`vm_init` Role)

此角色用于将一个原生 Debian 系统标准化为 Docker 运行时环境。

- **软件源配置**：强制覆盖 `/etc/apt/sources.list` 为清华源 (Tuna)，确保软件包下载速度 7。
- **Docker 安装**：直接调用官方脚本并指定 Aliyun 镜像源，保证安装过程不因网络超时中断 8。
- **数据盘挂载**：通过 UUID (`7ab6ec45...`) 挂载数据盘至 `/data`。这是为了确保容器数据最终落在 240G 的物理 SSD 上，而非系统盘 9。

## 4. 服务部署与监控体系 (`deploy_services` Role)

这是运维逻辑最复杂的部分，涉及权限管理、服务编排与监控自动发现。

### 4.1 权限与目录预埋

Docker 挂载目录的权限问题是常见痛点。Ansible 在部署前预先修正目录归属：
- **Prometheus**: uid `65534` (nobody) 10。
- **Grafana**: uid `472` 11。
- **目的**：防止容器启动后因 `Permission denied` 无法写入数据。

### 4.2 Prometheus 动态服务发现 (Service Discovery)

不再手动在 `prometheus.yml` 中填写每一个容器的 IP，而是配置 Prometheus 监听 Docker Socket。

- **机制**：`docker_sd_configs` 连接 `unix:///var/run/docker.sock` 12。    
- **筛选规则 (Relabeling)**：
    - 仅抓取带有 Label `monitor=true` 的容器 13。
    - 自动读取 Label `monitor_port` 作为抓取端口（如 cAdvisor 的 8080）14。
    - 使用容器名称 (`__meta_docker_container_name`) 作为监控实例名 15。

### 4.3 Docker Compose 模板化

使用 `docker-compose.yml.j2` 模板统一部署核心栈：

- **网关层**：Nginx Proxy Manager (80/443/81) 16。
- **监控层**：Prometheus + Grafana + cAdvisor + Node Exporter 17。
- **管理层**：Portainer 18。
- **配置更新**：当 `daemon.json` 发生变更（如镜像源更新）时，Ansible 会自动触发 Docker 服务重启 19。

## 5. 常用运维命令 (Playbook)

```Bash
# 1. 全量部署 (从 PVE 网络到 VM 业务)
ansible-playbook -i inventory.ini site.yml

# 2. 仅更新 PVE 网络配置 (如需新增端口映射)
ansible-playbook -i inventory.ini site.yml --tags "pve_host"

# 3. 仅更新 Docker 服务 (如修改了 docker-compose.yml)
ansible-playbook -i inventory.ini site.yml --tags "deploy_services"
```