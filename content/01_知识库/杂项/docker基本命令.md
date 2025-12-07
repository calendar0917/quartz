## docker

### 镜像

```docker
# 拉取镜像（缺省标签为latest）
docker pull 镜像名:标签

# 查看本地所有镜像
docker images

# 构建镜像（-t指定标签，.为Dockerfile所在目录）
docker build -t 镜像名:标签 构建目录

# 删除镜像（需先删基于该镜像的容器，-f强制删除）
docker rmi 镜像ID/镜像名

```

### 容器

```bash
# 创建并运行容器（核心参数：-d后台、-p端口映射、--name指定名称 -v 宿主机卷名:容器卷名）
docker run [参数] 镜像名

# 查看正在运行的容器
docker ps

# 查看所有容器（含已停止）
docker ps -a

# 启动已停止的容器
docker start 容器ID/容器名

# 优雅停止运行中的容器
docker stop 容器ID/容器名

# 强制停止容器（立即终止进程）
docker kill 容器ID/容器名

# 删除容器（需先停止，-f强制删除）
docker rm 容器ID/容器名

# 交互式进入容器终端
docker exec -it 容器ID/容器名 bash

# 查看容器日志（-f实时跟踪日志）
docker logs [参数] 容器ID/容器名

# 查看Docker系统配置、镜像源等信息
docker info

# 查看Docker版本
docker version
```

### docker compose

```bash
# 后台启动所有服务（-d后台运行）
docker compose up -d

# 仅停止所有服务（不删除容器/网络等资源）
docker compose stop

# 停止并删除容器、网络（-v同时删除数据卷）
docker compose down [参数]

# 查看Compose管理的容器状态
docker compose ps

# 重启所有服务
docker compose restart

# 构建/重建Compose中定义的镜像服务
docker compose build

# 查看服务日志（-f实时跟踪，可指定单个服务名）
docker compose logs [参数] [服务名]

# 交互式进入指定服务的容器终端
docker compose exec 服务名 bash
```