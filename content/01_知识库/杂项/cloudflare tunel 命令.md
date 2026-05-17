---
creation date: 2026-02-05 07:56
modification date: 2026-02-05 07:56
---
## 1. 核心：Cloudflare Tunnel (`cloudflared`)

这是你最需要熟悉的命令族。

### 基础命令
```bash
# 登录（第一次使用）
cloudflared login

# 创建隧道并生成配置文件
cloudflared tunnel --hostname yourdomain.com --url localhost:3000

# 在后台运行隧道（常用）
cloudflared tunnel --hostname yourdomain.com --url localhost:3000 --logfile /tmp/tunnel.log &
```

### 生产级配置（推荐）
创建配置文件 `~/.cloudflared/config.yml`：

```yaml
tunnel: your-tunnel-id
credentials-file: /root/.cloudflared/your-tunnel-id.json

ingress:
  # 映射到本地端口 3000
  - hostname: app.yourdomain.com
    service: http://localhost:3000
  
  # 映射到本地端口 8080  
  - hostname: blog.yourdomain.com
    service: http://localhost:8080
  
  # 映射多个端口（不同的子域名）
  - hostname: api.yourdomain.com
    service: http://localhost:8000
  
  # 默认规则（兜底）
  - service: http_status:404
```

启动：
```bash
# 启动所有配置的隧道
cloudflared tunnel run

# 指定配置文件
cloudflared tunnel --config /path/to/config.yml run

# 后台运行
nohup cloudflared tunnel run > /tmp/tunnel.log 2>&1 &
```

---

## 2. 本地开发常用命令

### 快速临时隧道（最简单）
```bash
# 单端口映射，生成临时域名
cloudflared tunnel --url localhost:3000
# 会显示类似 https://abc123.trycloudflare.com 的地址

# 映射到自定义子域名（需要域名已托管在 CF）
cloudflared tunnel --hostname dev.yourdomain.com --url localhost:3000
```

### 多端口批量映射脚本
创建一个脚本 `start-tunnels.sh`：

```bash
#!/bin/bash

# Web 应用
cloudflared tunnel --hostname app1.yourdomain.com --url localhost:3000 &
cloudflared tunnel --hostname app2.yourdomain.com --url localhost:8080 &
cloudflared tunnel --hostname docs.yourdomain.com --url localhost:3001 &

# 数据库（如果需要外网访问）
cloudflared tunnel --hostname db.yourdomain.com --url localhost:5432 &

wait
```

执行：
```bash
chmod +x start-tunnels.sh
./start-tunnels.sh
```

---

## 3. Cloudflare CLI 实用命令

### 账户和域名管理
```bash
# 查看当前账户信息
cloudflare

# 查看账户下的域名列表
cloudflare domains list

# 查看特定域名的 DNS 记录
cloudflare dns list yourdomain.com

# 添加 DNS 记录（如果需要手动管理）
cloudflare dns record add yourdomain.com A app.yourdomain.com 127.0.0.1
```

### 隧道管理
```bash
# 创建命名隧道
cloudflared tunnel create chat-server

# 手动添加 DNS 记录指向隧道
cloudflared tunnel route dns chat-server chat.calendar.de5.net

# 使用配置文件启动
cloudflared tunnel --config ~/.cloudflared/config.yml run

# 列出所有隧道
cloudflare tunnel list

# 查看隧道状态
cloudflare tunnel inspect your-tunnel-id

# 删除隧道
cloudflare tunnel delete your-tunnel-id
```

### Zero Trust 相关
```bash
# 登录 Zero Trust（管理页面）
cloudflared access login

# 生成访问令牌
cloudflare access token generate --save
```

---

## 4. 实际部署示例

### 场景1：本地多服务同时暴露
你的 `config.yml`：

```yaml
tunnel: dev-server
credentials-file: /root/.cloudflared/dev-server.json

ingress:
  # React 开发服务器
  - hostname: react.yourdomain.com
    service: http://localhost:5173
    
  # 后端 API 服务
  - hostname: api.yourdomain.com  
    service: http://localhost:3001
    
  # 文档服务
  - hostname: docs.yourdomain.com
    service: http://localhost:3000
    
  # 数据库管理工具
  - hostname: pma.yourdomain.com
    service: http://localhost:8080
    
  # 默认拒绝
  - service: http_status:404
```

启动：
```bash
# 启动开发环境
npm run dev &        # React (5173)
npm run api &        # Node.js (3001)
python docs serve &  # 文档 (3000)
docker run -p 8080:80 adminer &  # 数据库管理 (8080)

# 启动隧道
cloudflared tunnel run
```

### 场景2：移动端测试
在手机浏览器访问 `react.yourdomain.com`，效果和本地 `localhost:5173` 一样。

---

## 5. 故障排查命令

```bash
# 查看隧道日志（实时）
cloudflared tunnel --logfile - run

# 测试隧道连通性
curl -H "Host: app.yourdomain.com" http://localhost:8080

# DNS 解析测试
dig app.yourdomain.com

# 检查 CF 的边缘节点
curl -I https://app.yourdomain.com
```

---

## 6. 脚本化部署

创建 `deploy.sh`：
```bash
#!/bin/bash

echo "🚀 启动本地开发环境..."

# 启动本地服务
npm run dev &

# 等待服务启动
sleep 5

# 启动 Cloudflare 隧道
echo "🌐 启动 Cloudflare 隧道..."
cloudflared tunnel run

echo "✅ 开发环境已启动"
echo "📱 访问地址：https://react.yourdomain.com"
```

---

**总结重点：**
1. **开发期用：** `cloudflared tunnel --hostname xxx.yourdomain.com --url localhost:3000`
2. **生产期用：** 配置文件 + `cloudflared tunnel run`
3. **管理命令：** `cloudflared tunnel list/inspect/delete`

这样你就可以把本地任意端口通过 CF 安全地暴露到公网，无需配置路由器或防火墙。