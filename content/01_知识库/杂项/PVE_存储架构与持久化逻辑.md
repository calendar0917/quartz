---
creation date: 2025-12-27 18:55
modification date: 2025-12-27 18:55
---
## 1. 核心分层设计

为了确保数据在虚拟化层级中不丢失且读写高效，存储架构被划分为四个严密的层级。每一层都必须正确配置，数据才能最终落在物理硬盘上。

### 第一层：物理层与宿主机挂载 (PVE Host)

此层确保 240G SSD 被系统识别并持久化。
- **UUID 锁定**：必须通过 `by-uuid` 标识符挂载硬盘，防止因物理插槽变动导致盘符漂移（如 `sdb` 变为 `sdc`）。
- **挂载点**：`/mnt/pve/sdb-240g`。
- **配置方式**：通过 Ansible 的 `pve_storage` 角色写入 `/etc/fstab`。

### 第二层：虚拟化穿透 (LXC Bind Mount)

此层解决 LXC 容器（ID 100）与宿主机的文件共享问题。
- **映射指令**：`mp0` (Mount Point 0)。
- **配置命令**

```bash
# 将宿主机的 `docker_data` 目录穿透至容器内的 `/data` 路径。
pct set 100 -mp0 /mnt/pve/sdb-240g/docker_data,mp=/data
```

- **内核特权修复**    
    - **问题**：LXC 默认隔离内核日志，导致 cAdvisor 等监控工具无法启动。
    - 修复：在 /etc/pve/lxc/100.conf 中添加：

```
lxc.mount.entry: /dev/kmsg dev/kmsg none bind,optional,create=file。
```


### 第三层：应用持久化 (Docker Compose)

此层通过 Docker Volume 将应用数据写入已穿透的存储空间。

- **路径规则**：
    - **绝对路径**：用于访问系统资源（如 `/var/run/docker.sock`）。
    - **相对路径**：**严禁使用具名卷**。所有应用数据必须使用相对路径（如 `./grafana_data:/var/lib/grafana`）。
- **落盘逻辑**：由于 `docker-compose.yml` 存放在 `/data/core-services`，相对路径 `./` 会自动解析为宿主机的物理硬盘路径。

### 第四层：存储分工规范

严禁混用存储池，否则会导致系统盘爆满崩溃。

| **存储 ID**     | **物理介质**   | **角色定义** | **✅ 允许存放**                           | **❌ 禁止存放**          |
| ------------- | ---------- | -------- | ------------------------------------ | ------------------- |
| **local**     | 120G (sda) | 系统根目录    | PVE 系统日志, 极小的 ISO (如 Debian Netinst) | 虚拟机磁盘, 大型 ISO       |
| **local-lvm** | 120G (sda) | 快速存储     | **LXC 容器根文件系统**, Docker 镜像层          | Windows/Kali 等大型 VM |
| **Data-240G** | 240G (sdb) | 数据仓库     | **所有 VM 磁盘**, **ISO 镜像**, **备份文件**   | 无                   |