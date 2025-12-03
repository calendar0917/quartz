```bash
# 克隆安装脚本
git clone --branch master --depth 1 https://github.com/nelvko/clash-for-linux-install.git \
   && cd clash-for-linux-install \
   && sudo bash install.sh
# 检查安装状态
clashctl status
# 获取 web 控制台地址
clashctl ui
```