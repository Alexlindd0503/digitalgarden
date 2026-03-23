---
{"dg-publish":true,"permalink":"/66.归档发布/01.docker/Dockerfile使用指南/","dg-note-properties":{"时间":"2026-03-14"}}
---

#docker #java #最佳实践

```ad-summary
title: 总结

- 生产环境用多阶段构建，构建工具不进最终镜像；jar 已在 CI 构建好则用单阶段
- 基础镜像选 `eclipse-temurin:21-jre-alpine`，运行 Spring Boot 只需 JRE
- 容器内必须加 `-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0`，否则 JVM 不感知容器内存限制
```

## 1. Java 应用标准 Dockerfile

生产环境推荐用多阶段构建，构建工具不进最终镜像，体积小、安全：

```dockerfile
# 构建阶段
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
# 先复制 pom.xml，利用缓存，依赖没变就不重新下载
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# 运行阶段
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

如果 jar 包已经在 CI 里构建好了，直接用单阶段：

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 2. 常用 JVM 启动参数

容器环境下 JVM 默认不感知容器内存限制，需要显式配置：

```dockerfile
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

- `UseContainerSupport`：让 JVM 读取容器的 cgroup 内存限制（JDK 8u191+ 默认开启）
- `MaxRAMPercentage=75.0`：堆最多用容器内存的 75%，留余量给 Metaspace 和堆外内存
- `java.security.egd`：加快 SecureRandom 初始化，Spring Boot 启动更快

## 3. 基础镜像怎么选

`java:8` 这个镜像已经废弃，别用。现在主流选择：

| 镜像 | 大小 | 适用 |
|------|------|------|
| `eclipse-temurin:21-jre-alpine` | ~90MB | 生产首选，体积小 |
| `eclipse-temurin:21-jre` | ~250MB | 兼容性更好 |
| `amazoncorretto:21-alpine` | ~100MB | AWS 环境 |
| `eclipse-temurin:21-jdk-alpine` | ~190MB | 需要 JDK 工具时 |

运行 Spring Boot 只需要 JRE，不需要 JDK。

## 4. 几个关键指令

**COPY vs ADD**：优先用 `COPY`，`ADD` 只在需要自动解压 tar 包时用。

**CMD vs ENTRYPOINT**：Java 应用用 `ENTRYPOINT` 固定启动命令，`CMD` 可以作为默认参数被覆盖。

**RUN 合并**：多条命令合成一个 `RUN`，减少镜像层数：

```dockerfile
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

**WORKDIR**：用 `WORKDIR /app` 设置工作目录，别用 `RUN cd /app`（不生效）。

## 5. .dockerignore

```
.git
target/
*.md
*.log
.env
.idea
```

把 `target/` 排除掉，避免把本地编译产物带进构建上下文。

## 6. 常用构建命令

```bash
# 构建
docker build -t myapp:1.0 .

# 指定 Dockerfile
docker build -f Dockerfile.prod -t myapp:1.0 .

# 不用缓存（依赖有问题时用）
docker build --no-cache -t myapp:1.0 .
```
