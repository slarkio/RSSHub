# RSSHub Docker 部署文档

本文档针对 fork 的 RSSHub 项目的 Docker 部署方案，满足特定需求的定制化部署。

## 🚀 快速开始（TL;DR）

如果你只想快速部署，按顺序执行以下命令：

```bash
# 1. 清理环境
docker-compose down
docker-compose -f docker-compose.local.yml down

# 2. 构建镜像（关键：使用 china.Dockerfile）
docker build -f china.Dockerfile -t rsshub-local:china .

# 3. 启动服务
docker-compose -f docker-compose.local.yml up -d

# 4. 验证部署
curl http://localhost:1200/healthz
```

**⚠️ 重要**：必须使用 `-f china.Dockerfile` 才能应用清华镜像源优化！

## 特殊需求说明

### 1. Fork 项目部署
- **需求**：使用自己 fork 的 RSSHub 项目，而非官方镜像
- **解决方案**：本地构建 Docker 镜像，确保使用最新的 fork 代码

### 2. 启用 Puppeteer 功能
- **需求**：支持需要 JavaScript 渲染的网站（如 B站、微博等）
- **解决方案**：使用 Browserless 服务提供独立的浏览器环境

### 3. 数据持久化
- **需求**：缓存数据、日志、配置文件需要持久化保存
- **解决方案**：挂载本地目录到容器，确保数据不丢失

### 4. 国内网络优化
- **需求**：构建过程使用国内镜像源提高下载速度
- **解决方案**：使用清华大学镜像源替换默认的 Debian 和 npm 源

## 部署架构

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   RSSHub        │    │   Browserless   │    │     Redis       │
│  (本地构建)      │◄──►│   (Chrome)      │    │   (缓存存储)     │
│  Port: 1200     │    │  Port: 3000     │    │  Port: 6379     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   数据持久化     │
                    │   ./data/       │
                    └─────────────────┘
```

## 文件结构

```
RSSHub/
├── docker-compose.local.yml     # 本地部署配置
├── china.Dockerfile            # 使用国内镜像源的构建文件
├── data/                       # 数据持久化目录
│   ├── rsshub/
│   │   ├── logs/              # RSSHub 应用日志
│   │   ├── config/            # RSSHub 配置文件
│   │   └── puppeteer/         # Puppeteer 缓存
│   ├── browserless/
│   │   ├── downloads/         # 浏览器下载文件
│   │   ├── fonts/            # 自定义字体
│   │   └── sessions/         # 浏览器会话数据
│   └── redis/
│       └── redis.conf        # Redis 配置文件
└── volumes/
    └── redis-data/           # Redis 数据存储
```

## 部署步骤

⚠️ **重要提醒**：必须按照以下顺序执行，不能跳过任何步骤！

### 步骤 0：环境准备

```bash
# 停止可能运行的旧服务
docker-compose down
docker-compose -f docker-compose.local.yml down

# 清理旧镜像（可选，如果需要完全重新开始）
docker rmi rsshub-local rsshub-local:china 2>/dev/null || true
```

### 步骤 1：构建本地镜像 🔧

**⚠️ 关键步骤：必须先构建镜像才能启动服务！**

```bash
# 使用 china.Dockerfile 构建镜像（使用清华镜像源）
docker build -f china.Dockerfile -t rsshub-local:china .

# 验证镜像构建成功
docker images | grep rsshub-local
# 应该看到类似输出：
# rsshub-local   china   <IMAGE_ID>   <TIME>   <SIZE>
```

**构建过程说明**：
- 构建时间：约 5-15 分钟（取决于网络状况）
- 使用清华大学镜像源，大幅提升下载速度
- 镜像大小：约 2-3GB

### 步骤 2：验证镜像构建

```bash
# 检查镜像详情
docker inspect rsshub-local:china

# 验证镜像中的配置（可选）
docker run --rm rsshub-local:china npm config get registry
# 应该输出：https://registry.npmmirror.com
```

### 步骤 3：启动服务 🚀

**只有在步骤 1 成功完成后才能执行此步骤！**

```bash
# 启动所有服务（RSSHub + Browserless + Redis）
docker-compose -f docker-compose.local.yml up -d

# 查看服务状态
docker-compose -f docker-compose.local.yml ps
# 应该看到 3 个服务都是 "Up" 状态

# 查看启动日志（重要：检查是否有错误）
docker-compose -f docker-compose.local.yml logs -f rsshub
```

### 步骤 4：验证部署 ✅

```bash
# 等待服务完全启动（约 30-60 秒）
sleep 60

# 检查服务健康状态
curl http://localhost:1200/healthz
# 应该返回：{"status":"ok"}

# 测试基础功能
curl http://localhost:1200
# 应该返回 RSSHub 主页

# 测试 Puppeteer 功能（需要更长时间）
curl http://localhost:1200/bilibili/user/dynamic/2267573
```

## 常见错误和解决方案

### ❌ 错误 1：直接运行 docker-compose 但镜像不存在
```bash
# 错误信息：
# ERROR: pull access denied for rsshub-local:china, repository does not exist

# 解决方案：必须先构建镜像
docker build -f china.Dockerfile -t rsshub-local:china .
```

### ❌ 错误 2：使用了错误的 Dockerfile
```bash
# 错误做法：
docker build -t rsshub-local:china .  # 使用默认 Dockerfile

# 正确做法：
docker build -f china.Dockerfile -t rsshub-local:china .  # 使用清华源
```

### ❌ 错误 3：忘记指定配置文件
```bash
# 错误做法：
docker-compose up -d  # 使用默认配置

# 正确做法：
docker-compose -f docker-compose.local.yml up -d  # 使用本地配置
```

## 服务配置详情

### RSSHub 服务
- **镜像**：`rsshub-local:china`（本地构建）
- **端口**：1200
- **环境变量**：
  - `NODE_ENV=production`
  - `CACHE_TYPE=redis`
  - `REDIS_URL=redis://redis:6379/`
  - `PUPPETEER_WS_ENDPOINT=ws://browserless:3000`
- **资源限制**：内存 2GB，共享内存 1GB

### Browserless 服务
- **镜像**：`browserless/chrome`
- **端口**：3000（可选暴露，用于调试）
- **配置**：
  - 最大并发会话：10
  - 连接超时：60秒
  - 启用广告拦截和隐身模式
- **资源限制**：内存 2GB，CPU 2核

### Redis 服务
- **镜像**：`redis:alpine`
- **端口**：6379（内部）
- **持久化**：AOF + RDB 双重保障
- **配置**：最大内存 256MB，LRU 淘汰策略

## 管理命令

### 服务管理
```bash
# 停止服务
docker-compose -f docker-compose.local.yml down

# 重启服务
docker-compose -f docker-compose.local.yml restart

# 更新服务（重新构建镜像）
docker build -f china.Dockerfile -t rsshub-local:china .
docker-compose -f docker-compose.local.yml up -d --force-recreate rsshub
```

### 日志查看
```bash
# 查看所有服务日志
docker-compose -f docker-compose.local.yml logs

# 查看特定服务日志
docker-compose -f docker-compose.local.yml logs rsshub
docker-compose -f docker-compose.local.yml logs browserless
docker-compose -f docker-compose.local.yml logs redis

# 实时跟踪日志
docker-compose -f docker-compose.local.yml logs -f --tail=100
```

### 数据备份
```bash
# 备份所有持久化数据
tar -czf rsshub-backup-$(date +%Y%m%d-%H%M%S).tar.gz data/

# 备份 Redis 数据
docker-compose -f docker-compose.local.yml exec redis redis-cli BGSAVE

# 恢复数据
tar -xzf rsshub-backup-20250101-120000.tar.gz
docker-compose -f docker-compose.local.yml restart
```

## 故障排除

### 常见问题

1. **镜像构建失败**
   ```bash
   # 清理构建缓存
   docker builder prune
   
   # 重新构建
   docker build -f china.Dockerfile -t rsshub-local:china . --no-cache
   ```

2. **Puppeteer 功能异常**
   ```bash
   # 检查 Browserless 服务状态
   curl http://localhost:3000/pressure
   
   # 重启 Browserless
   docker-compose -f docker-compose.local.yml restart browserless
   ```

3. **缓存问题**
   ```bash
   # 清理 Redis 缓存
   docker-compose -f docker-compose.local.yml exec redis redis-cli FLUSHALL
   
   # 重启 Redis
   docker-compose -f docker-compose.local.yml restart redis
   ```

### 性能优化

1. **调整资源限制**
   - 根据服务器配置调整 `mem_limit` 和 `cpu_count`
   - 监控资源使用情况：`docker stats`

2. **缓存优化**
   - 调整 Redis 内存限制和淘汰策略
   - 监控缓存命中率

3. **网络优化**
   - 使用国内 Docker Hub 镜像
   - 配置代理服务器（如需要）

## 更新升级

### 代码更新
```bash
# 1. 停止服务
docker-compose -f docker-compose.local.yml down

# 2. 拉取最新代码
git pull origin main

# 3. 重新构建镜像（关键：必须使用 china.Dockerfile）
docker build -f china.Dockerfile -t rsshub-local:china .

# 4. 启动服务
docker-compose -f docker-compose.local.yml up -d
```

### 依赖更新
```bash
# 更新 package 依赖后需要重新构建
npm update
docker build -f china.Dockerfile -t rsshub-local:china . --no-cache
```

## 安全注意事项

1. **网络安全**
   - 生产环境建议不暴露 Browserless 端口
   - 使用防火墙限制访问

2. **数据安全**
   - 定期备份持久化数据
   - 监控磁盘空间使用

3. **容器安全**
   - 定期更新基础镜像
   - 使用非 root 用户运行（如可能）

---

*最后更新：2025-08-01*
*适用版本：RSSHub v1.0.0+*