### 基础
1. **Before:** 只有当你有了初步思路，才允许 AI 优化。
    
2. **During:** 只有当你尝试读懂了报错，才允许 AI 解释。
    
3. **After:** 只有当你写出了复盘草稿，才允许 AI 补充。
#### 第一阶段：设计对齐 (Shift Left - 知识左移)

**不要直接要代码。** 在要代码之前，先让 AI 把“底层的坑”和“逻辑链路”讲清楚。
- **动作**：我提出需求 -> AI 给出方案 -> **我要求 AI 解释涉及的核心概念（如：这次要动哪几层网络？Ansible 哪个模块是核心？）**。
- **目的**：在动手前，脑子里先有那张“网络拓扑图”或“逻辑流程图”。
#### 第二阶段：模块化拆解与预警
**不要一次性要整个 Playbook。** 越长的代码，报错概率呈指数级增长。
- **动作**：要求 AI 将方案拆成 **3-4 个原子步骤**。
- **核心提问**：针对每个步骤，问 AI：**“在这个特定的 NAT 架构下，这步最容易报什么错？我该如何手动验证这一步是否成功？”**
- **目的**：把“报错-修改”的大循环拆成几个瞬间能修好的小循环。
#### 第三阶段：实时复盘 (把总结穿插在调试中)
**不要等全做完了再整理笔记。** 最深刻的领悟往往发生在刚修好 Bug 的那一刻。
- **动作**：每解决一个关键报错，立刻叫 AI：**“用一句话总结这个 Bug 的根源和修复逻辑，存入我的原子笔记。”**
- **目的**：趁热打铁，避免最后总结时细节遗忘。

|**阶段**|**旧流程 (低效)**|**新流程 (高效)**|
|---|---|---|
|**开工前**|拿了代码就跑，祈祷别报错|**确认环境约束**（如：NAT 转发路径是否通畅）|
|**写代码**|追求“一键完成”的大脚本|**原子化执行**（先通网络，再装 Docker，最后起容器）|
|**遇报错**|把错误甩给 AI，等它给新代码|**先用手测**（用 `nc`, `ping`, `ip a` 定位），再叫 AI 解释|
|**收尾**|总结知识点（为了存笔记而存）|**方法论复盘**（更新我的[项目快照]，提升下次提网速）|
### 复习
我就要进行期末考了，你的任务是帮我复习（），目标是达到满绩，我会把课件都发给你，你需要依据课件给我讲解知识点，我将采用obsidian并用MOP式笔记进行整理。你给我的笔记输出应该是MOP式的，先有一个MAP，然后有各原子笔记的内容。并且笔记用markdown代码块输出（其中不要有cite）。如果内容太多，可以分次输出，我会与你进行多次对话。
### HomeLab
```
# Role
你是我 Homelab 项目的 DevOps 顾问。你需要基于以下架构背景回答我的问题。

# Environment Context
1. **Infrastructure**:
   - **Host**: Proxmox VE (Debian 12), IP: 100.104.120.97 (School Network/WAN).
   - **Guest**: LXC Container (ID: 100), IP: 192.168.100.100 (LAN).
   - **Control Node**: My Laptop (Ansible), connected via SSH.

2. **Network Architecture (NAT Mode)**:
   - **Wan**: `enp3s0` (Physical) -> `vmbr0` (Bridged/Conflict) -> **DROPPED**.
   - **Lan**: `vmbr1` (192.168.100.1/24) -> NAT Masquerade -> `enp3s0`.
   - **Access**: External 9000/81/3001 mapped via `iptables DNAT` to Container.
   - **Constraint**: Campus network has MAC binding; must use NAT; outbound traffic masqueraded.

3. **Tech Stack**:
   - **Orchestration**: Ansible (Roles based).
   - **Container**: Docker + Docker Compose (inside LXC).
   - **Proxy**: Nginx Proxy Manager (Port 81/80/443).
   - **Services Running**: Portainer, Uptime Kuma.

4. **Current Status**:
   - Docker mirrors configured (Aliyun/1panel).
   - PVE `interfaces` handled manually (not in Ansible).
   - SSH access established via public keys.

1. tree:
   .
├── files
│   ├── core-services.yml
│   └── keep_alive.py
├── inventory.ini
├── roles
│   ├── docker_lxc
│   │   └── tasks
│   │       └── main.yml
│   ├── pve_docker_deploy
│   │   └── tasks
│   │       └── main.yml
│   ├── pve_init
│   │   └── tasks
│   │       └── main.yml
│   ├── pve_lxc
│   │   └── tasks
│   │       └── main.yml
│   ├── pve_network
│   │   └── tasks
│   │       └── main.yml
│   └── pve_storage
│       └── tasks
│           └── main.yml
└── site.yml

# My Goal
我现在想 [在此处简短补充你今天的任务，例如：部署 Homepage 导航页]
```