---
{"dg-publish":true,"permalink":"/03-方法卡/Dockerfile最佳实践/","tags":["dockerfile","最佳实践","镜像优化","方法卡","tech/docker"]}
---

## 1. 核心原则
编写高效、安全、可维护的 Dockerfile，构建体积小、层数少、构建快的 Docker 镜像。

## 2. 镜像优化

### 2.1. 选择合适的基础镜像
```dockerfile
# ❌ 不推荐：完整系统镜像
FROM ubuntu:latest

# ✅ 推荐：精简版镜像
FROM ubuntu:20.04-slim

# ✅ 更好：Alpine 镜像（最小）
FROM alpine:3.18

# ✅ 最佳：专用语言镜像
FROM node:18-alpine
FROM python:3.11-slim
```

**选择建议：**
- 优先使用官方镜像
- 选择带 `-slim` 或 `-alpine` 后缀的精简版
- 避免使用 `latest` 标签，指定明确版本

### 2.2. 合并 RUN 指令减少层数
```dockerfile
# ❌ 不推荐：多层
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get clean

# ✅ 推荐：单层
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        vim && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**优势：**
- 减少镜像层数
- 降低镜像体积
- 清理缓存更彻底

### 2.3. 利用构建缓存
```dockerfile
# ✅ 推荐：先复制依赖文件
COPY package.json package-lock.json ./
RUN npm install

# 后复制源代码（变化频繁）
COPY . .
```

**原则：**
- 变化少的指令放前面
- 变化频繁的指令放后面
- 充分利用层缓存机制

### 2.4. 使用多阶段构建
```dockerfile
# 构建阶段
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# 运行阶段
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

**优势：**
- 分离构建和运行环境
- 最终镜像不包含构建工具
- 大幅减小镜像体积

### 2.5. 使用 .dockerignore
```dockerignore
# .dockerignore 文件
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.DS_Store
*.md
```

**作用：**
- 排除不必要的文件
- 加快构建速度
- 减小上下文大小

## 3. 安全实践

### 3.1. 使用非 root 用户
```dockerfile
# 创建用户
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

# 切换用户
USER appuser

# 设置工作目录权限
WORKDIR /app
RUN chown -R appuser:appuser /app
```

### 3.2. 不在镜像中存储敏感信息
```dockerfile
# ❌ 不推荐：硬编码密钥
ENV API_KEY=secret123

# ✅ 推荐：运行时传入
# docker run -e API_KEY=secret123 myapp
```

### 3.3. 使用固定版本
```dockerfile
# ❌ 不推荐：不确定版本
FROM node:latest
RUN apt-get install -y curl

# ✅ 推荐：明确版本
FROM node:18.17.0-alpine3.18
RUN apk add --no-cache curl=8.2.1-r0
```

## 4. 性能优化

### 4.1. 合理使用 COPY 和 ADD
```dockerfile
# ✅ 推荐：使用 COPY
COPY package.json .
COPY src/ ./src/

# ⚠️ 特殊场景：ADD 自动解压
ADD archive.tar.gz /app/
```

**区别：**
- `COPY`: 简单复制文件
- `ADD`: 支持 URL 和自动解压，但不推荐常用

### 4.2. 清理临时文件
```dockerfile
RUN apt-get update && \
    apt-get install -y build-essential && \
    # 编译操作 && \
    apt-get purge -y build-essential && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### 4.3. 优化 COPY 顺序
```dockerfile
# 按变化频率排序
COPY package.json .          # 很少变化
RUN npm install              # 依赖变化时才重新执行
COPY config/ ./config/       # 偶尔变化
COPY src/ ./src/             # 经常变化
```

## 5. 可维护性

### 5.1. 使用 LABEL 添加元数据
```dockerfile
LABEL maintainer="dev@example.com"
LABEL version="1.0.0"
LABEL description="应用描述"
LABEL org.opencontainers.image.source="https://github.com/user/repo"
```

### 5.2. 合理使用 ARG 和 ENV
```dockerfile
# ARG：构建时变量
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

# ENV：运行时环境变量
ENV NODE_ENV=production
ENV PORT=3000
```

### 5.3. 明确指定 WORKDIR
```dockerfile
# ✅ 推荐：使用 WORKDIR
WORKDIR /app
COPY . .

# ❌ 不推荐：使用 RUN cd
RUN cd /app && npm install
```

## 6. 完整示例

### 6.1. Node.js 应用
```dockerfile
# 多阶段构建
FROM node:18-alpine AS builder
WORKDIR /app

# 复制依赖文件
COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# 复制源码并构建
COPY . .
RUN npm run build

# 生产镜像
FROM node:18-alpine
LABEL maintainer="dev@example.com"

# 创建非 root 用户
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

WORKDIR /app

# 从构建阶段复制文件
COPY --from=builder --chown=appuser:appuser /app/dist ./dist
COPY --from=builder --chown=appuser:appuser /app/node_modules ./node_modules
COPY --chown=appuser:appuser package.json ./

# 切换用户
USER appuser

# 暴露端口
EXPOSE 3000

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1

# 启动命令
CMD ["node", "dist/index.js"]
```

### 6.2. Java 应用
```dockerfile
# 构建阶段
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# 运行阶段
FROM eclipse-temurin:17-jre-alpine
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

WORKDIR /app
COPY --from=builder --chown=appuser:appuser /app/target/*.jar app.jar

USER appuser
EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 7. 检查清单

### 7.1. 构建前检查
- [ ] 选择了合适的基础镜像
- [ ] 创建了 .dockerignore 文件
- [ ] 合并了可以合并的 RUN 指令
- [ ] 按变化频率排序了指令

### 7.2. 安全检查
- [ ] 使用了非 root 用户
- [ ] 没有硬编码敏感信息
- [ ] 使用了明确的版本号
- [ ] 清理了临时文件和缓存

### 7.3. 优化检查
- [ ] 使用了多阶段构建
- [ ] 镜像层数合理（< 20 层）
- [ ] 镜像体积可接受
- [ ] 添加了健康检查

## 8. 常用命令

### 8.1. 构建镜像
```bash
# 基本构建
docker build -t myapp:1.0 .

# 使用构建参数
docker build --build-arg NODE_VERSION=18 -t myapp:1.0 .

# 不使用缓存
docker build --no-cache -t myapp:1.0 .
```

### 8.2. 分析镜像
```bash
# 查看镜像层
docker history myapp:1.0

# 查看镜像大小
docker images myapp:1.0

# 扫描安全漏洞
docker scan myapp:1.0
```

## 9. 相关概念
- [[01-概念卡/Docker镜像构建原理\|Docker镜像构建原理]] - 分层原理
- [[01-概念卡/OverlayFS文件系统\|OverlayFS文件系统]] - 存储机制
- [[03-方法卡/Docker离线镜像处理方法\|Docker离线镜像处理方法]] - 镜像操作
- [[多阶段构建详解\|多阶段构建详解]] - 高级技巧

## 10. 参考资源
- [Docker 官方最佳实践](https://docs.docker.com/develop/dev-best-practices/)
- [Dockerfile 参考文档](https://docs.docker.com/engine/reference/builder/)

