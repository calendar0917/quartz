# system服务配置

系统服务可以用于服务的系统级管理，从而实现自启动、后台运行、权限管理等等。

原理其实也比较简单，就是 init fork 出一个子进程就可以了，然后父进程 waitpid 来实现管理。

基本的配置是：

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application Service
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/myapp --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5
TimeoutStartSec=30
TimeoutStopSec=30

# 安全配置
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/myapp /var/lib/myapp

# 资源限制
LimitNOFILE=65536
MemoryMax=512M
CPUQuota=200%

[Install]
WantedBy=multi-user.target
```