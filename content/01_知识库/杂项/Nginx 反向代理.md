## 1. 原理机制
客户端向 Nginx（公网）发起请求，Nginx 作为**反向代理服务器**将请求转发给内网的实际服务，并将响应返回给客户端。
* **客户端视角**：只与 Nginx 交互，不知道后端存在。
* **优势**：隐藏后端架构、负载均衡、统一端口管理。

## 2. 配置文件结构
通常在 `/etc/nginx/conf.d/` 下新建 `.conf` 文件，而不是直接修改主文件。

> [!warning] 端口冲突警告
> 如果 Nginx 主配置 `/etc/nginx/nginx.conf` 已经监听了 80 端口，新建的配置若也监听 80 会导致启动失败。需先注释掉默认的 server 块。

```nginx
# /etc/nginx/conf.d/reverse-proxy.conf

server {
    listen 80;
    server_name 8.137.xx.xx;  # 填写公网 IP 或域名

    location / {
        # 核心：转发地址（内网服务的 IP:Port）
        proxy_pass [http://127.0.0.1:8001](http://127.0.0.1:8001); 
        
        # 关键头信息传递（防止后端应用获取不到真实 IP）
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## 3. 常用操作

- **测试配置**：`nginx -t`
    
- **重载服务**：`nginx -s reload` 或 `systemctl reload nginx`