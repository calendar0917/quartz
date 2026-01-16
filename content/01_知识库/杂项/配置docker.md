## CentOS7

```bash
# 查看是否已经安装：
docker --version
# 若已安装，删除：
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine \
                  docker-ce
# 安装yum依赖工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# 添加docker官方仓库
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# 阿里云：
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 安装
sudo yum install -y docker-ce docker-ce-cli containerd.io
```

## Ubuntu

```bash
# 移除无效密钥文件
rm -f /etc/apt/trusted.gpg.d/docker.gpg
# 更换方式获取密钥
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
# 替换docker源
# 删除原有Docker源
rm -f /etc/apt/sources.list.d/docker.list
# 添加阿里云Docker源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/trusted.gpg.d/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
# 更新源并安装docker
apt update && apt install -y docker-ce docker-ce-cli containerd.io
# 自启动
sudo systemctl start docker
sudo systemctl enable docker
```

## 添加docker镜像源
```bash
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://alzgoonw.mirror.aliyuncs.com",
    "https://docker.m.daocloud.io",
    "https://dockerhub.icu",
    "https://docker.anyhub.us.kg",
    "https://docker.1panel.live"
  ]
}

EOF
```

## LXC 中使用docker

参考：[SOLVED - Docker inside LXC (net.ipv4.ip_unprivileged_port_start error) | Proxmox Support Forum](https://forum.proxmox.com/threads/docker-inside-lxc-net-ipv4-ip_unprivileged_port_start-error.175437/)

版本选择依据

1. **避开有 bug 的版本**：论坛明确 `1.7.28-2` 及更高版本（如 `1.7.29-1`、`2.1.5-1`）会触发 LXC 中 Docker 修改 `net.ipv4.ip_unprivileged_port_start` 的权限错误，必须选择**低于 1.7.28-2** 的版本。
2. **选择最接近的稳定版本**：`1.7.28-1~ubuntu.24.04~noble` 是 `1.7.28-2` 的前一个版本，仅差小版本修复，功能完整且无该兼容性 bug，比更旧的 `1.7.27-1`/`1.7.26-1` 更推荐（减少版本回退带来的其他兼容性问题）。

如果还是不行，尝试[[命令行配置clash]]
