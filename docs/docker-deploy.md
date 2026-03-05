# Docker 部署指南

使用 Docker 一键部署 OpenClaw Gateway，无需安装 Node.js，开箱即用。

## 目录

- [快速开始](#快速开始)
- [Docker Compose 部署](#docker-compose-部署)
- [配置说明](#配置说明)
- [模型配置](#模型配置)
- [数据持久化](#数据持久化)
- [网络与端口](#网络与端口)
- [反向代理](#反向代理)
- [常用命令](#常用命令)
- [升级](#升级)
- [常见问题](#常见问题)

---

## 快速开始

**一条命令启动：**

```bash
docker run -d \
  --name openclaw \
  --restart unless-stopped \
  -p 18789:18789 \
  -v openclaw-data:/root/.openclaw \
  node:22-slim \
  sh -c "npm install -g @qingchencloud/openclaw-zh --registry https://registry.npmmirror.com && openclaw gateway start"
```

启动后访问 `http://你的服务器IP:18789`。

> ⚠️ 首次启动需要下载安装 OpenClaw，可能需要 1-2 分钟。后续重启会直接启动。

---

## Docker Compose 部署

推荐使用 Docker Compose，配置更清晰，管理更方便。

### 1. 创建项目目录

```bash
mkdir -p ~/openclaw && cd ~/openclaw
```

### 2. 创建配置文件

```bash
cat > openclaw.json << 'EOF'
{
  "mode": "local",
  "tools": {
    "profile": "full",
    "sessions": {
      "visibility": "all"
    }
  },
  "gateway": {
    "port": 18789,
    "bind": "lan",
    "auth": {
      "mode": "token",
      "token": "your-secret-token-change-me"
    }
  },
  "models": {
    "providers": {
      "deepseek": {
        "baseUrl": "https://api.deepseek.com/v1",
        "apiKey": "sk-your-api-key",
        "models": [
          { "id": "deepseek-chat", "name": "DeepSeek V3" },
          { "id": "deepseek-reasoner", "name": "DeepSeek R1" }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "deepseek/deepseek-chat"
      }
    }
  }
}
EOF
```

> 记得修改 `token` 和 `apiKey` 为你的实际值。

### 3. 创建 docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  openclaw:
    image: node:22-slim
    container_name: openclaw
    restart: unless-stopped
    ports:
      - "18789:18789"
    volumes:
      - ./openclaw.json:/root/.openclaw/openclaw.json
      - openclaw-data:/root/.openclaw
    command: >
      sh -c "
        npm install -g @qingchencloud/openclaw-zh --registry https://registry.npmmirror.com 2>/dev/null;
        openclaw gateway start
      "
    environment:
      - NODE_ENV=production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:18789/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

volumes:
  openclaw-data:
EOF
```

### 4. 启动

```bash
docker compose up -d
```

### 5. 查看日志

```bash
docker compose logs -f
```

看到 `Gateway listening on http://0.0.0.0:18789` 就说明启动成功了。

---

## 使用自定义 Dockerfile（推荐生产环境）

上面的方式每次容器重建都要重新 `npm install`，生产环境建议构建自定义镜像：

### Dockerfile

```bash
cat > Dockerfile << 'EOF'
FROM node:22-slim

# 安装 OpenClaw
RUN npm install -g @qingchencloud/openclaw-zh \
    --registry https://registry.npmmirror.com \
    && npm cache clean --force

# 安装 curl 用于健康检查
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

# 配置目录
RUN mkdir -p /root/.openclaw
VOLUME /root/.openclaw

EXPOSE 18789

CMD ["openclaw", "gateway", "start"]
EOF
```

### docker-compose.yml（使用自定义镜像）

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  openclaw:
    build: .
    container_name: openclaw
    restart: unless-stopped
    ports:
      - "18789:18789"
    volumes:
      - ./openclaw.json:/root/.openclaw/openclaw.json
      - openclaw-data:/root/.openclaw
    environment:
      - NODE_ENV=production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:18789/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

volumes:
  openclaw-data:
EOF
```

```bash
# 构建并启动
docker compose up -d --build
```

这样容器重启时直接启动 Gateway，无需重复安装。

---

## 配置说明

### 核心配置项

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `mode` | 运行模式，必须设置 | `"local"` |
| `tools.profile` | Agent 工具权限 | `"full"` |
| `tools.sessions.visibility` | 会话可见性 | `"all"` |
| `gateway.port` | Gateway 监听端口 | `18789` |
| `gateway.bind` | 绑定范围 | `"lan"` |
| `gateway.auth.mode` | 认证方式 | `"token"` |
| `gateway.auth.token` | 访问密钥 | 无 |

### bind 选项

| 值 | 说明 |
|------|------|
| `"loopback"` | 仅容器内部访问（127.0.0.1），Docker 环境下**不要用这个** |
| `"lan"` | 所有网卡（0.0.0.0），Docker 环境**必须用这个** |

> ⚠️ Docker 容器内必须用 `"lan"`，否则端口映射无法工作。

---

## 模型配置

在 `openclaw.json` 的 `models.providers` 中添加服务商。以下是常见服务商示例：

### DeepSeek

```json
{
  "models": {
    "providers": {
      "deepseek": {
        "baseUrl": "https://api.deepseek.com/v1",
        "apiKey": "sk-xxxx",
        "models": [
          { "id": "deepseek-chat", "name": "DeepSeek V3" },
          { "id": "deepseek-reasoner", "name": "DeepSeek R1" }
        ]
      }
    }
  }
}
```

### OpenAI

```json
{
  "openai": {
    "baseUrl": "https://api.openai.com/v1",
    "apiKey": "sk-xxxx",
    "models": [
      { "id": "gpt-4o", "name": "GPT-4o" },
      { "id": "gpt-4o-mini", "name": "GPT-4o Mini" }
    ]
  }
}
```

### Claude（通过兼容 API）

```json
{
  "claude": {
    "baseUrl": "https://your-claude-proxy.com/v1",
    "apiKey": "sk-xxxx",
    "models": [
      { "id": "claude-sonnet-4-20250514", "name": "Claude Sonnet 4" }
    ]
  }
}
```

### Ollama（本地模型）

```json
{
  "ollama": {
    "baseUrl": "http://host.docker.internal:11434/v1",
    "apiKey": "ollama",
    "models": [
      { "id": "qwen2.5:14b", "name": "Qwen 2.5 14B" },
      { "id": "deepseek-r1:8b", "name": "DeepSeek R1 8B" }
    ]
  }
}
```

> 注意 Ollama 在 Docker 里要用 `host.docker.internal` 访问宿主机，Linux 下需要加 `--add-host=host.docker.internal:host-gateway`。

### Ollama + Docker 特殊配置

如果 Ollama 也运行在同一台机器上：

```yaml
services:
  openclaw:
    # ... 其他配置 ...
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

或者让 OpenClaw 和 Ollama 共享 Docker 网络：

```yaml
services:
  openclaw:
    # ... 其他配置 ...
    network_mode: host  # 直接使用宿主机网络，Ollama 用 localhost:11434

  # 或者 Ollama 也在 Compose 里
  ollama:
    image: ollama/ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
```

---

## 数据持久化

OpenClaw 的所有数据在 `/root/.openclaw` 目录：

| 路径 | 内容 |
|------|------|
| `openclaw.json` | 全局配置 |
| `agents/` | Agent 数据 |
| `memories/` | 记忆文件 |
| `backups/` | 配置备份 |

通过 Docker Volume 或 bind mount 持久化：

```yaml
# Docker Volume（推荐，Docker 自动管理）
volumes:
  - openclaw-data:/root/.openclaw

# Bind mount（挂载本地目录，方便直接编辑）
volumes:
  - ./data:/root/.openclaw
```

### 备份

```bash
# 备份整个数据目录
docker cp openclaw:/root/.openclaw ./openclaw-backup-$(date +%Y%m%d)

# 仅备份配置
docker cp openclaw:/root/.openclaw/openclaw.json ./openclaw.json.bak
```

---

## 网络与端口

### 修改端口

```yaml
# docker-compose.yml
ports:
  - "8080:18789"  # 宿主机 8080 → 容器 18789
```

或者同时修改配置文件中的端口：

```json
{
  "gateway": {
    "port": 8080
  }
}
```

```yaml
ports:
  - "8080:8080"
```

### 使用 host 网络

```yaml
services:
  openclaw:
    network_mode: host
    # 不需要 ports 映射，直接用宿主机端口
```

适用于需要直接访问宿主机服务（如 Ollama）的场景。

---

## 反向代理

### Nginx + Docker Compose

```yaml
version: '3.8'

services:
  openclaw:
    build: .
    container_name: openclaw
    restart: unless-stopped
    volumes:
      - ./openclaw.json:/root/.openclaw/openclaw.json
      - openclaw-data:/root/.openclaw
    # 不暴露端口到外部，只在内部网络

  nginx:
    image: nginx:alpine
    container_name: openclaw-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - openclaw

volumes:
  openclaw-data:
```

**nginx.conf：**

```nginx
server {
    listen 80;
    server_name ai.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name ai.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    location / {
        proxy_pass http://openclaw:18789;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }
}
```

---

## 常用命令

```bash
# 启动
docker compose up -d

# 停止
docker compose down

# 重启
docker compose restart

# 查看日志
docker compose logs -f

# 查看最近 100 行日志
docker compose logs --tail 100

# 进入容器
docker exec -it openclaw sh

# 查看容器状态
docker compose ps

# 重新构建并启动（修改了 Dockerfile 后）
docker compose up -d --build
```

---

## 升级

### 使用自定义 Dockerfile

```bash
cd ~/openclaw

# 重新构建镜像（会拉取最新版 OpenClaw）
docker compose build --no-cache

# 重启
docker compose up -d
```

### 使用 node 基础镜像（容器内升级）

```bash
# 进入容器
docker exec -it openclaw sh

# 升级
npm install -g @qingchencloud/openclaw-zh --registry https://registry.npmmirror.com

# 退出容器
exit

# 重启容器
docker restart openclaw
```

---

## 多实例部署

在同一台机器运行多个 OpenClaw 实例（不同端口、不同配置）：

```yaml
version: '3.8'

services:
  openclaw-main:
    build: .
    container_name: openclaw-main
    restart: unless-stopped
    ports:
      - "18789:18789"
    volumes:
      - ./config-main.json:/root/.openclaw/openclaw.json
      - main-data:/root/.openclaw

  openclaw-dev:
    build: .
    container_name: openclaw-dev
    restart: unless-stopped
    ports:
      - "18790:18789"
    volumes:
      - ./config-dev.json:/root/.openclaw/openclaw.json
      - dev-data:/root/.openclaw

volumes:
  main-data:
  dev-data:
```

---

## 常见问题

### 容器启动后无法访问

1. 确认 `gateway.bind` 是 `"lan"`（不是 `"loopback"`）
2. 检查端口映射：`docker compose ps`
3. 检查容器日志：`docker compose logs`
4. 检查防火墙是否放行端口

### 首次启动很慢

首次启动需要 `npm install` 安装 OpenClaw，耗时 1-2 分钟。建议使用[自定义 Dockerfile](#使用自定义-dockerfile推荐生产环境) 方案，构建时安装，启动时直接运行。

### 容器重启后配置丢失

确保使用了 volume 持久化：

```yaml
volumes:
  - openclaw-data:/root/.openclaw
```

或 bind mount 挂载本地目录。

### 连接 Ollama 报错

Docker 容器内 `localhost` 是容器自己，不是宿主机。访问宿主机的 Ollama 需要：

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

然后 baseUrl 填 `http://host.docker.internal:11434/v1`。

### 内存占用

OpenClaw Gateway 本身约 50-100MB。如果同时运行 Ollama 等本地模型，建议 4GB+ 内存。

### 时区问题

```yaml
services:
  openclaw:
    environment:
      - TZ=Asia/Shanghai
```

---

## 完整示例

生产就绪的 docker-compose.yml：

```yaml
version: '3.8'

services:
  openclaw:
    build: .
    container_name: openclaw
    restart: unless-stopped
    ports:
      - "18789:18789"
    volumes:
      - ./openclaw.json:/root/.openclaw/openclaw.json
      - openclaw-data:/root/.openclaw
    environment:
      - NODE_ENV=production
      - TZ=Asia/Shanghai
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:18789/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  openclaw-data:
```
