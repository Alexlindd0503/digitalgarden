---
{"dg-publish":true,"permalink":"/01-概念卡/Docker构建缓存机制/","tags":["docker","构建缓存","性能优化","buildkit","概念卡"]}
---


## 1. 核心概念
Docker 构建缓存是一种优化机制，通过复用之前构建的镜像层，避免重复执行相同的指令，从而大幅提升构建速度。

## 2. 工作原理

### 2.1. 缓存匹配规则
Docker 按顺序检查每条指令，判断是否可以使用缓存：

```dockerfile
FROM node:18-alpine          # 检查基础镜像
WORKDIR /app                 # 检查工作目录
COPY package.json .          # 检查文件内容（checksum）
RUN npm install              # 检查命令字符串
COPY . .                     # 检查文件内容
RUN npm run build            # 检查命令字符串
```

### 2.2. 缓存判断逻辑

#### 2.2.1. FROM 指令
- 检查基础镜像是否存在于本地
- 镜像标签和摘要必须完全匹配

#### 2.2.2. COPY/ADD 指令
- 计算文件内容的 **checksum**（校验和）
- 比较文件元数据（修改时间、权限等）
- 任何文件变化都会导致缓存失效

#### 2.2.3. RUN 指令
- 比较命令字符串是否完全相同
- **不执行命令**，仅比较文本
- 即使命令结果不同，字符串相同就使用缓存

#### 2.2.4. 其他指令
- ENV, LABEL, EXPOSE 等：比较指令内容
- CMD, ENTRYPOINT：仅修改元数据，不影响缓存

## 3. 缓存失效规则

### 3.1. 链式失效
```dockerfile
FROM node:18-alpine          # ✅ 缓存命中
COPY package.json .          # ✅ 缓存命中
RUN npm install              # ✅ 缓存命中
COPY src/app.js .            # ❌ 文件变化，缓存失效
RUN npm run build            # ❌ 上层失效，后续全部失效
CMD ["node", "app.js"]       # ❌ 上层失效，后续全部失效
```

**关键点：**
- 一旦某层缓存失效，后续所有层都失效
- 即使后续指令未变化，也必须重新执行

### 3.2. 常见失效场景

#### 3.2.1. 文件内容变化
```dockerfile
COPY package.json .          # package.json 修改
RUN npm install              # 缓存失效，重新安装
```

#### 3.2.2. 命令字符串变化
```dockerfile
# 原命令
RUN apt-get update && apt-get install -y curl

# 修改后（即使功能相同）
RUN apt-get update && apt-get install -y curl vim
# 缓存失效
```

#### 3.2.3. 构建参数变化
```dockerfile
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}    # NODE_VERSION 变化导致缓存失效
```

## 4. 缓存优化策略

### 4.1. 按变化频率排序
```dockerfile
# ✅ 推荐：变化少的在前
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./        # 依赖文件（很少变化）
RUN npm install              # 安装依赖（很少重新执行）
COPY . .                     # 源码（经常变化）
RUN npm run build            # 构建（经常重新执行）

# ❌ 不推荐：源码在前
FROM node:18-alpine
COPY . .                     # 源码变化
RUN npm install              # 每次都重新安装
```

**效果对比：**
- 优化前：源码变化 → 重新安装依赖（耗时 2-5 分钟）
- 优化后：源码变化 → 仅重新构建（耗时 10-30 秒）

### 4.2. 分离依赖和源码
```dockerfile
# 依赖层（缓存稳定）
COPY package.json package-lock.json ./
RUN npm ci

# 配置层（偶尔变化）
COPY tsconfig.json .eslintrc.js ./

# 源码层（频繁变化）
COPY src ./src
RUN npm run build
```

### 4.3. 合并命令但保持缓存
```dockerfile
# ❌ 不推荐：过度合并
RUN apt-get update && \
    apt-get install -y curl vim git && \
    curl -o app.tar.gz https://example.com/app.tar.gz && \
    tar -xzf app.tar.gz
# 任何一步变化都要全部重新执行

# ✅ 推荐：合理分层
RUN apt-get update && \
    apt-get install -y curl vim git
RUN curl -o app.tar.gz https://example.com/app.tar.gz
RUN tar -xzf app.tar.gz
# 下载失败只需重新执行下载步骤
```

### 4.4. 使用 .dockerignore
```dockerignore
# 排除不影响构建的文件
node_modules
.git
.env
*.md
.DS_Store
coverage/
```

**作用：**
- 减少构建上下文大小
- 避免无关文件变化导致缓存失效
- 提升 COPY 指令的缓存命中率

## 5. BuildKit 缓存增强

### 5.1. 启用 BuildKit
```bash
# 临时启用
export DOCKER_BUILDKIT=1
docker build -t myapp .

# 永久启用（Docker 配置）
{
  "features": {
    "buildkit": true
  }
}
```

### 5.2. 外部缓存源
```bash
# 从注册表拉取缓存
docker build \
  --cache-from myregistry/myapp:cache \
  -t myapp:latest .

# 推送缓存到注册表
docker build \
  --cache-to type=registry,ref=myregistry/myapp:cache \
  -t myapp:latest .
```

### 5.3. 内联缓存
```bash
# 构建时嵌入缓存元数据
docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t myapp:latest .

# 后续构建使用内联缓存
docker build \
  --cache-from myapp:latest \
  -t myapp:v2 .
```

### 5.4. 本地缓存目录
```bash
# 使用本地目录缓存
docker build \
  --cache-to type=local,dest=/tmp/docker-cache \
  -t myapp:latest .

# 从本地目录加载缓存
docker build \
  --cache-from type=local,src=/tmp/docker-cache \
  -t myapp:latest .
```

## 6. RUN 缓存挂载

### 6.1. 缓存包管理器下载
```dockerfile
# Node.js npm 缓存
RUN --mount=type=cache,target=/root/.npm \
    npm install

# Python pip 缓存
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Go mod 缓存
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# Maven 缓存
RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:go-offline
```

**优势：**
- 跨构建共享下载缓存
- 即使层缓存失效，也不重新下载
- 大幅减少网络请求

### 6.2. 缓存编译产物
```dockerfile
# Go 编译缓存
RUN --mount=type=cache,target=/root/.cache/go-build \
    go build -o app .

# Rust 编译缓存
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release
```

## 7. 调试缓存

### 7.1. 查看缓存使用情况
```bash
# 构建时显示详细信息
docker build --progress=plain -t myapp .

# 查看构建历史
docker history myapp:latest

# 查看缓存层
docker image inspect myapp:latest
```

### 7.2. 禁用缓存
```bash
# 完全禁用缓存
docker build --no-cache -t myapp .

# 从指定层开始禁用缓存
docker build --cache-from myapp:base -t myapp .
```

### 7.3. 清理缓存
```bash
# 清理构建缓存
docker builder prune

# 清理所有未使用的缓存
docker builder prune -a

# 查看缓存使用情况
docker system df
```

## 8. 实战案例

### 8.1. 案例 1：Node.js 应用优化
```dockerfile
# ❌ 优化前：每次源码变化都重装依赖
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install              # 耗时 3 分钟
RUN npm run build
# 构建时间：~3.5 分钟

# ✅ 优化后：利用缓存
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci                   # 首次 3 分钟，后续 10 秒
COPY . .
RUN npm run build
# 构建时间：首次 3.5 分钟，后续 30 秒
```

### 8.2. 案例 2：Python 应用优化
```dockerfile
# ✅ 多层缓存策略
FROM python:3.11-slim
WORKDIR /app

# 系统依赖层（很少变化）
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc

# Python 依赖层（偶尔变化）
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# 应用代码层（频繁变化）
COPY . .
CMD ["python", "app.py"]
```

### 8.3. 案例 3：Go 应用优化
```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app

# 依赖下载层
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# 编译层
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o app .

FROM alpine:3.18
COPY --from=builder /app/app /usr/local/bin/
CMD ["app"]
```

## 9. 缓存性能对比

### 9.1. 构建时间对比
| 场景 | 无优化 | 基础优化 | BuildKit 缓存 |
|------|--------|----------|---------------|
| 首次构建 | 5 分钟 | 5 分钟 | 5 分钟 |
| 依赖未变 | 5 分钟 | 30 秒 | 10 秒 |
| 仅源码变化 | 5 分钟 | 30 秒 | 10 秒 |
| 完全相同 | 5 分钟 | 5 秒 | 1 秒 |

### 9.2. 缓存命中率
- **无优化**: ~20% 缓存命中
- **基础优化**: ~70% 缓存命中
- **BuildKit + 挂载缓存**: ~90% 缓存命中

## 10. 最佳实践

### 10.1. ✅ 推荐做法
1. 按文件变化频率排序指令
2. 分离依赖安装和代码复制
3. 使用 .dockerignore 排除无关文件
4. 启用 BuildKit 和缓存挂载
5. 使用外部缓存源（CI/CD 环境）
6. 合理使用多阶段构建

### 10.2. ❌ 避免做法
1. 在 RUN 中使用随机数或时间戳
2. 过早复制整个项目目录
3. 频繁修改基础镜像版本
4. 在 Dockerfile 中使用 `apt-get upgrade`
5. 忽略 .dockerignore 文件

## 11. 常见问题

### 11.1. Q1: 为什么缓存没有生效？
- 检查文件是否真的未变化（包括权限、时间戳）
- 确认 .dockerignore 是否正确配置
- 查看是否使用了 `--no-cache` 参数

### 11.2. Q2: 如何强制更新某一层？
```dockerfile
# 使用 ARG 强制刷新
ARG CACHE_BUST=1
RUN apt-get update && apt-get install -y curl
```

### 11.3. Q3: CI/CD 环境如何利用缓存？
```yaml
# GitHub Actions 示例
- name: Build with cache
  uses: docker/build-push-action@v4
  with:
    cache-from: type=registry,ref=myapp:cache
    cache-to: type=registry,ref=myapp:cache,mode=max
```

## 12. 相关概念
- [[01-概念卡/Docker镜像构建原理\|Docker镜像构建原理]] - 分层存储
- [[03-方法卡/Dockerfile最佳实践\|Dockerfile最佳实践]] - 构建优化
- [[03-方法卡/Docker多阶段构建详解\|Docker多阶段构建详解]] - 高级技巧
- [[01-概念卡/OverlayFS文件系统\|OverlayFS文件系统]] - 存储机制

## 13. 参考资源
- [Docker 构建缓存官方文档](https://docs.docker.com/build/cache/)
- [BuildKit 缓存后端](https://docs.docker.com/build/cache/backends/)

