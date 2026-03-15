---
{"dg-publish":true,"permalink":"/66.归档发布/00-0其他/SpringBoot3 Web容器配置指南/"}
---

#springboot #tomcat #undertow #性能调优

## 1. 先选容器

Spring Boot 3 内置三种 Web 容器，默认是 Tomcat，也可以换成 Undertow。

| 容器 | 特点 | 适合场景 |
|------|------|---------|
| Tomcat | 成熟稳定，生态好，默认选择 | 大部分业务场景 |
| Undertow | 更轻量，内存占用低，非阻塞 IO | 高并发 API、对内存敏感 |
| Netty | 响应式编程专用（WebFlux） | 流式、响应式场景 |

没有特殊需求就用 Tomcat，想要更低内存占用可以换 Undertow。两者在普通业务场景下性能差距感知不明显，换容器不是性能优化的首选手段。

Java 21 + Spring Boot 3.2 之后，开启虚拟线程是更值得考虑的优化方向（见第 4 节）。

## 2. Tomcat 配置

### 2.1 基本配置

```yaml
server:
  port: 8080
  tomcat:
    threads:
      max: 400          # 最大工作线程数，默认 200
      min-spare: 20     # 最小空闲线程数
    max-connections: 8192   # 最大连接数，默认 8192
    accept-count: 100       # 等待队列长度，超出直接拒绝
    connection-timeout: 20000  # 连接超时，单位 ms
```

注意：Spring Boot 3.x 线程数配置改到了 `threads.max`，老的 `max-threads` 写法不生效。

### 2.2 线程数怎么定？

不要套公式，从保守值开始压测。一般参考：

| 服务器规格 | max 推荐起点 | 说明 |
|-----------|------------|------|
| 2 核 4G | 100~200 | 先压测再调 |
| 4 核 8G | 200~400 | 先压测再调 |
| 8 核 16G | 400~800 | 先压测再调 |

线程不是越多越好，线程过多会导致频繁上下文切换，CPU 调度开销反而成为瓶颈。每个线程还会占用约 1MB 栈内存，800 个线程光栈就 800MB。

`max-connections` 可以远大于 `max-threads`，因为大部分连接处于等待 IO 状态，只有真正在处理请求时才占用线程。

### 2.3 IO 密集 vs CPU 密集

IO 密集型（大量数据库查询、外部接口调用）：线程大部分时间在等待，可以适当多配线程，让 CPU 有事可做。

CPU 密集型（大量计算、图像处理）：线程数接近 CPU 核心数就够了，多了反而增加切换开销。

## 3. 换成 Undertow

排除 Tomcat 依赖，引入 Undertow：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

配置：

```yaml
server:
  undertow:
    threads:
      io: 4          # IO 线程数，建议等于 CPU 核心数
      worker: 128    # 工作线程数，处理业务逻辑
    buffer-size: 1024
    direct-buffers: true  # 使用堆外内存，减少 GC 压力
```

Undertow 的 IO 线程负责接收连接和读写数据，worker 线程负责处理业务逻辑。IO 线程数一般等于 CPU 核心数就够了。

## 4. 虚拟线程（Java 21+）

Spring Boot 3.2 支持虚拟线程，开启方式：

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

开启后 Tomcat 会用虚拟线程处理每个请求，创建和切换开销极低，可以轻松支撑数万并发。IO 密集型场景下效果最明显，`max-threads` 的限制基本可以忽略了。

虚拟线程不适合 CPU 密集型任务，那种场景还是用平台线程。

## 5. 监控线程池

```java
@Component
@Slf4j
public class TomcatThreadPoolMonitor {

    @Autowired
    private TomcatWebServer tomcatWebServer;

    @Scheduled(fixedRate = 30000)
    public void monitor() {
        Connector connector = tomcatWebServer.getTomcat().getConnector();
        AbstractProtocol<?> protocol = (AbstractProtocol<?>) connector.getProtocolHandler();
        log.info("Tomcat 线程池 - 当前: {}, 活跃: {}, 最大: {}, 连接数: {}",
            protocol.getCurrentThreadCount(),
            protocol.getCurrentThreadsBusy(),
            protocol.getMaxThreads(),
            protocol.getConnectionCount());
    }
}
```

关注几个指标：
- 活跃线程数 / 最大线程数 > 80%，说明线程池快撑不住了
- 连接数持续接近 `max-connections`，考虑扩容或排查慢请求
- `evicted_keys` 频繁增长（Redis 侧），说明上游压力大

## 相关链接

- [[66.归档发布/02.编码相关/JVM性能调优方法\|JVM性能调优方法]]
