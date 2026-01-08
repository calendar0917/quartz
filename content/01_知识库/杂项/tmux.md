---
creation date: 2025-12-06 11:41
modification date: 2025-12-06 11:41
---
### 结构
```text
Server (后台守护进程)
└── Session (会话)  <-- 对应一个具体的“项目”或“任务上下文”
    └── Window (窗口)  <-- 对应浏览器的一个“标签页” (Tab)
        └── Pane (窗格)  <-- 对应屏幕上的一个“终端切片”
```
### 基础命令
操作：

```bash
# 启动会话，不要直接tmux启动匿名会话
tmux new -s name

# 切割窗格 ctrl+b+
% # 左右
"" # 上下

# 焦点 ctrl+b+
方向键

# 脱离 ctrl+b+
d # 进程并没有死

# 接管
tmux a -t name # a是attach的缩写

# 销毁 pane 内输入
exit
ctrl + d
```

会话管理：

```bash
# 列出后台所有存活会话
tmux ls 
# 在不同会话间切换
tmux switch -t name
# 改名
tmux rename-session -t old new
# 关闭 session
tmux kill-session -t name
# 关闭整个 tmux 服务器
tmux kill-server
# 显示会话列表
pre + s

```

窗口管理：

```bash
# pre + ...
# 新建窗口
c
# 重命名
，
# 下一个
n
# 上一个
p
# 第 n 号
num
# 展开所有窗口
w
```

窗格管理：

```bash
# pre + ...
# 最大化
z
# 关闭
x
# 交换位置
{}
```
### 自定义
```bash
vim ~/.tmux.conf
```

```bash
# --- 基础设置 ---
# 开启鼠标支持（这是为了让你平滑过渡，后期建议全键盘）
# 允许鼠标选择窗格、调节大小、滚动历史
set -g mouse on

# 将前缀键修改为 Ctrl + a (致敬 GNU Screen，且更符合人体工学)
# 注意：这会覆盖 Bash 的 "移到行首" 快捷键，看你取舍
# 如果你想保留 Ctrl+b，请注释掉下面两行
set -g prefix C-a
unbind C-b
bind C-a send-prefix

# --- 像 Vim 一样移动 ---
# 使用 vim 模式的快捷键在 pane 之间移动
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# --- 视觉优化 ---
# 状态栏颜色（极客黑绿）
set -g status-bg black
set -g status-fg green
# 开启 256 色支持，否则你的 Vim 配色会很奇怪
set -g default-terminal "screen-256color"

```

在 tmux 窗口中，先按下 `Ctrl+b` 指令前缀，然后按下系统指令:，进入到命令模式后输入 `source-file ~/.tmux.conf`，回车后生效。