> **核心定义**：Dockerfile 不仅仅是安装脚本，它是对 **UnionFS（联合文件系统）** 的分层描述。每一条指令（Instruction）都在只读层（Read-only Layer）之上叠加新的变更。

## 核心机制：分层与缓存 (Layers & Cache)

*   **机制**：Docker Daemon 会**缓存**每一行指令构建出的 Layer。如果 `instruction` 和 `context file` 没有变化，直接复用缓存。
*   **优化策略**：**将变更最频繁的指令放在最后**。
    *   ❌ **错误**：先 COPY 代码，再安装依赖（代码一改，依赖缓存失效，每次都要重新 pip install）。
    *   ✅ **正确**：先 COPY 依赖描述文件（package.json/requirements.txt），安装依赖，**最后** COPY 源代码。

```df
# 缓存命中率极高
COPY requirements.txt .
RUN pip install -r requirements.txt
# 经常变化，放在最后
COPY src/ .
```

## 关键指令解析

### FROM: 选择基座
*   **安全视角**：基座决定了攻击面（Attack Surface）的大小。
*   **选择优先级**：`distroless` (Google无Shell镜像) > `alpine` (musl libc) > `slim` (精简Debian) > `latest` (完整OS)。
*   *注意：Alpine 使用 musl libc，可能导致 C 扩展库（如 numpy/pandas）编译问题，需权衡构建成本。*

### RUN: 执行与层合并
*   **原子化**：每一个 `RUN` 都会生成新的一层。
*   **清理原则**：在同一个 `RUN` 指令中安装包并清理缓存，防止临时文件被持久化到中间层。
    ```dockerfile
    # 推荐写法：使用 && 连接，并清理 apt 缓存
    RUN apt-get update && apt-get install -y \
        gcc \
        && rm -rf /var/lib/apt/lists/*
    ```

### CMD vs ENTRYPOINT: PID 1 的控制权
*   **容器即进程**：容器内的 PID 1 进程负责接收内核信号（SIGTERM/SIGINT）。
*   **Shell 模式 vs Exec 模式**：
    *   `CMD "python app.py"` (Shell模式): PID 1 是 `/bin/sh`，应用是子进程，**收不到停止信号**，导致无法优雅退出。
    *   `CMD ["python", "app.py"]` (Exec模式): **推荐**。PID 1 直接是 python 进程，能处理信号。
*   **组合拳**：用 `ENTRYPOINT` 定义“不可变”的启动命令，用 `CMD` 定义“可覆盖”的默认参数。

## 多阶段构建 (Multi-stage Build)

解决 **"构建环境" vs "运行环境"** 的矛盾。

*   **场景**：Go/C++/Java/Python(带C扩展) 编译。
*   **原理**：利用 `AS` 别名，在一个 Dockerfile 中定义多个流，最后只保留 Runtime 必需的文件。
*   **价值**：
    1.  **极度精简**：产物从 1GB -> 50MB。
    2.  **安全**：生产环境不包含编译器（GCC/Make）、源码和调试工具，黑客进入容器后难以利用。

```df
# Stage 1: Builder (包含 GCC, Headers 等)
FROM python:3.9-slim AS builder
WORKDIR /app
COPY requirements.txt .
# 编译并安装到用户目录
RUN pip install --user -r requirements.txt

# Stage 2: Runtime (只有 Python 解释器)
FROM python:3.9-slim
WORKDIR /app
# 仅复制 Stage 1 的构建产物（Python包）
COPY --from=builder /root/.local /root/.local
COPY app.py .
# 确保 path 包含用户目录
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

## 安全加固 CheckList

1.  **非 Root 运行**：永远不要以 Root 身份运行应用。
    ```dockerfile
    RUN useradd -m appuser
    USER appuser
    ```
2.  **敏感信息**：**严禁**在 Dockerfile 中使用 `ENV` 存储密码/Key（`docker history` 可见）。应在运行时通过 `-e` 或 Volume 挂载传入。
3.  **.dockerignore**：必须配置。防止 `.git`、`__pycache__`、`.env` 等敏感或无用文件被打入 Context，导致镜像膨胀或泄露。