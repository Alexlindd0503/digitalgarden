---
{"dg-publish":true,"permalink":"/03-方法卡/docker/Docker多阶段构建详解/","tags":["多阶段构建","镜像优化","高级技巧","方法卡","tech/docker"]}
---

## 1. 核心概念
多阶段构建（Multi-stage Build）允许在一个 Dockerfile 中使用多个 `FROM` 指令，每个 `FROM` 开始一个新的构建阶段，最终镜像只包含最后阶段的内容。

## 2. 基本原理

### 2.1. 传统构建 vs 多阶段构建
```dockerfile
# ❌ 传统方式：包含构建工具
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install        # 包含 devDependencies
COPY . .
RUN npm run build      # 构建工具留在镜像中
CMD ["node", "dist/index.js"]
# 结果：镜像体积 ~1GB

# ✅ 多阶段构建：分离构建和运行
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
# 结果：镜像体积 ~150MB
```

## 3. 高级技巧

### 3.1. 命名构建阶段
```dockerfile
# 为每个阶段命名，提高可读性
FROM node:18-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine AS production
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

**优势：**
- 语义清晰，易于维护
- 可以选择性复制特定阶段的产物
- 便于调试和测试

### 3.2. 使用外部镜像作为阶段
```dockerfile
# 从外部镜像复制文件
FROM nginx:alpine
COPY --from=node:18-alpine /usr/local/bin/node /usr/local/bin/
COPY --from=myregistry/common-libs:1.0 /libs /app/libs
COPY dist/ /usr/share/nginx/html/
```

**应用场景：**
- 复用已有镜像的工具或库
- 避免重复构建公共组件
- 组合不同来源的资源

### 3.3. 构建时选择目标阶段
```dockerfile
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS development
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

FROM base AS builder
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

**使用方式：**
```bash
# 构建开发镜像
docker build --target development -t myapp:dev .

# 构建生产镜像
docker build --target production -t myapp:prod .

# 默认构建最后阶段
docker build -t myapp:latest .
```

### 3.4. 并行构建优化
```dockerfile
# 依赖阶段（可并行）
FROM node:18-alpine AS deps-prod
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS deps-dev
WORKDIR /app
COPY package*.json ./
RUN npm ci

# 构建阶段（依赖 deps-dev）
FROM deps-dev AS builder
COPY . .
RUN npm run build

# 测试阶段（依赖 deps-dev，可与 builder 并行）
FROM deps-dev AS tester
COPY . .
RUN npm run test

# 生产阶段
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=deps-prod /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

**优势：**
- Docker 自动识别可并行的阶段
- 充分利用多核 CPU
- 缩短总构建时间

### 3.5. 条件构建（使用 ARG）
```dockerfile
ARG BUILD_ENV=production

FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS deps-dev
RUN npm install

FROM base AS deps-prod
RUN npm ci --only=production

FROM deps-${BUILD_ENV} AS deps
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=deps /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

**使用方式：**
```bash
# 开发环境构建
docker build --build-arg BUILD_ENV=dev -t myapp:dev .

# 生产环境构建
docker build --build-arg BUILD_ENV=prod -t myapp:prod .
```

### 3.6. 跨平台构建
```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.21-alpine AS builder
ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

WORKDIR /app
COPY go.* ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -o app .

FROM alpine:3.18
COPY --from=builder /app/app /usr/local/bin/
CMD ["app"]
```

**使用方式：**
```bash
# 构建多平台镜像
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

## 4. 实战案例

### 4.1. 案例 1：前端应用（React/Vue）
```dockerfile
# 构建阶段
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 生产阶段：使用 Nginx
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**效果：**
- 构建阶段：~1GB（包含 Node.js 和构建工具）
- 最终镜像：~20MB（仅 Nginx + 静态文件）

### 4.2. 案例 2：Go 应用（静态编译）
```dockerfile
# 构建阶段
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

# 运行阶段：使用 scratch（最小镜像）
FROM scratch
COPY --from=builder /app/app /app
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/app"]
```

**效果：**
- 构建阶段：~500MB
- 最终镜像：~10MB（仅二进制文件）

### 4.3. 案例 3：Python 应用（依赖分离）
```dockerfile
# 依赖构建阶段
FROM python:3.11-slim AS deps
WORKDIR /app
RUN pip install --no-cache-dir poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt -o requirements.txt --without-hashes

# 应用构建阶段
FROM python:3.11-slim AS builder
WORKDIR /app
COPY --from=deps /app/requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# 运行阶段
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

### 4.4. 案例 4：Java 应用（分层 JAR）
```dockerfile
# 构建阶段
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests
RUN java -Djarmode=layertools -jar target/*.jar extract

# 运行阶段：分层复制
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

**优势：**
- 利用 Spring Boot 分层特性
- 依赖层变化少，缓存效果好
- 应用层变化频繁，独立更新

### 4.5. 案例 5：多服务构建
```dockerfile
# 共享基础阶段
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci

# API 服务构建
FROM base AS api-builder
COPY apps/api ./apps/api
COPY libs ./libs
RUN npm run build:api

# Web 服务构建
FROM base AS web-builder
COPY apps/web ./apps/web
COPY libs ./libs
RUN npm run build:web

# API 生产镜像
FROM node:18-alpine AS api
WORKDIR /app
COPY --from=api-builder /app/dist/api ./
CMD ["node", "main.js"]

# Web 生产镜像
FROM nginx:alpine AS web
COPY --from=web-builder /app/dist/web /usr/share/nginx/html
```

**使用方式：**
```bash
# 构建 API 服务
docker build --target api -t myapp-api:latest .

# 构建 Web 服务
docker build --target web -t myapp-web:latest .
```

## 5. 调试技巧

### 5.1. 查看中间阶段
```bash
# 构建到指定阶段
docker build --target builder -t myapp:builder .

# 运行中间阶段进行调试
docker run -it myapp:builder sh
```

### 5.2. 保留构建缓存
```bash
# 使用 BuildKit 缓存
export DOCKER_BUILDKIT=1
docker build --cache-from myapp:cache -t myapp:latest .

# 推送缓存
docker build --cache-to type=registry,ref=myapp:cache .
```

### 5.3. 分析镜像层
```bash
# 查看每个阶段的大小
docker history myapp:latest

# 使用 dive 工具分析
dive myapp:latest
```

## 6. 性能优化

### 6.1. 优化复制顺序
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app

# 先复制依赖文件（变化少）
COPY package*.json ./
RUN npm ci

# 再复制配置文件（偶尔变化）
COPY tsconfig.json ./
COPY .eslintrc.js ./

# 最后复制源码（经常变化）
COPY src ./src
RUN npm run build
```

### 6.2. 并行安装依赖
```dockerfile
FROM node:18-alpine AS deps-runtime
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS deps-build
WORKDIR /app
COPY package*.json ./
RUN npm ci

# 两个阶段可并行执行
```

### 6.3. 使用构建参数优化
```dockerfile
ARG NODE_ENV=production
ARG NPM_REGISTRY=https://registry.npmjs.org

FROM node:18-alpine AS builder
ARG NPM_REGISTRY
RUN npm config set registry ${NPM_REGISTRY}
WORKDIR /app
COPY package*.json ./
RUN npm ci
```

## 7. 最佳实践总结

### 7.1. ✅ 推荐做法
1. 为每个阶段命名，提高可读性
2. 将构建工具和运行环境分离
3. 使用最小化的基础镜像
4. 合理利用构建缓存
5. 按变化频率排序复制操作
6. 使用 `--target` 构建不同环境镜像

### 7.2. ❌ 避免做法
1. 在最终镜像中保留构建工具
2. 复制不必要的文件到生产镜像
3. 使用过多的构建阶段（影响可读性）
4. 忽略 .dockerignore 文件
5. 在多个阶段重复相同操作

## 8. 相关概念
- [[03-方法卡/docker/Dockerfile最佳实践\|Dockerfile最佳实践]] - 基础优化
- [[01-概念卡/docker/Docker镜像构建原理\|Docker镜像构建原理]] - 分层原理
- [[01-概念卡/docker/OverlayFS文件系统\|OverlayFS文件系统]] - 存储机制
- [[01-概念卡/docker/Docker构建缓存机制\|Docker构建缓存机制]] - 缓存优化

## 9. 参考资源
- [Docker 多阶段构建官方文档](https://docs.docker.com/build/building/multi-stage/)
- [BuildKit 高级特性](https://docs.docker.com/build/buildkit/)

#docker #多阶段构建 #镜像优化 #高级技巧
