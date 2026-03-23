---
{"dg-publish":true,"permalink":"/66.归档发布/03.缓存/07-Redis高频面试题/","dg-note-properties":{"时间":"2026-03-23"}}
---

#redis #面试 #高频

```ad-summary
title: 这篇笔记讲什么

- Redis 为什么快：内存 + 单线程 + IO 多路复用
- 过期策略：惰性删除 + 定期删除
- 与 Memcached 区别：数据结构、持久化、集群
- 线程模型：6.0 前单线程，6.0 后多线程 IO
```

## 1. Redis 为什么这么快

**答**：内存存储 + 单线程避免锁竞争 + IO 多路复用。具体的数据结构可以看 [[66.归档发布/03.缓存/00-Redis基础概念\|00-Redis基础概念]]，持久化和高可用方案看 [[66.归档发布/03.缓存/01-Redis持久化机制\|01-Redis持久化机制]]和 [[66.归档发布/03.缓存/02-Redis高可用与复制\|02-Redis高可用与复制]]。

| 因素 | 说明 |
|------|------|
| 内存存储 | 数据在内存中，读写速度是磁盘的 10 万倍 |
| 单线程模型 | 避免多线程上下文切换和锁竞争 |
| IO 多路复用 | 用 epoll 同时监听多个连接，非阻塞 |
| 高效数据结构 | SDS、压缩列表、跳表等针对场景优化 |

## 2. Redis 过期策略

**问**：key 设置了过期时间，到时间一定会删除吗？

**答**：不一定。Redis 用两种策略配合删除：

| 策略 | 做法 | 特点 |
|------|------|------|
| 惰性删除 | 访问 key 时检查是否过期 | 不消耗 CPU，但可能漏掉过期 key |
| 定期删除 | 定期扫描一部分 key | 折中，每秒 10 次，每次 20 个 key |

如果两种都没删掉，内存淘汰策略会兜底。

## 3. Redis 和 Memcached 的区别

| | Redis | Memcached |
|--|-------|-----------|
| 数据结构 | String、List、Hash、Set、ZSet 等 | 只有 String |
| 持久化 | 支持 RDB、AOF | 不支持 |
| 集群 | 原生支持 Cluster | 需要客户端实现 |
| 线程模型 | 单线程（6.0 后多线程 IO） | 多线程 |
| 内存管理 | 自己管理 | 用 Slab 分配器 |
| 适用场景 | 复杂数据结构、持久化需求 | 纯缓存、简单 KV |

**简单场景用 Memcached，复杂场景用 Redis。**

## 4. Redis 线程模型

**问**：Redis 是单线程还是多线程？

**答**：
- **6.0 之前**：完全单线程，包括网络 IO 和命令执行
- **6.0 之后**：网络 IO 多线程，命令执行还是单线程

6.0 引入多线程 IO 的原因：
- 网络 IO 是瓶颈，特别是大 value 读写
- 命令执行还是单线程，保证线程安全
- 默认关闭，需要配置 `io-threads 4`

## 5. 如何保证缓存与数据库一致性

**问**：更新数据库后，缓存怎么同步？

**答**：

| 方案 | 一致性 | 性能 | 适用场景 |
|------|--------|------|----------|
| 延迟双删 | 最终一致 | 快 | 大部分场景 |
| 分布式锁 | 强一致 | 慢 | 金融、库存 |
| Canal 监听 Binlog | 最终一致 | 中 | 不想改业务代码 |

详见 [[66.归档发布/03.缓存/04-Redis缓存问题与解决方案\|04-Redis缓存问题与解决方案]]。

## 6. 缓存雪崩、击穿、穿透的区别

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 雪崩 | 大量 key 同时过期 | 随机过期时间 + 多级缓存 |
| 击穿 | 热点 key 过期 | 互斥锁 / 永不过期 |
| 穿透 | 查不存在的数据 | 布隆过滤器 / 缓存空值 |

详见 [[66.归档发布/03.缓存/04-Redis缓存问题与解决方案\|04-Redis缓存问题与解决方案]]。

## 7. Redis 持久化方式怎么选

**问**：RDB 和 AOF 选哪个？

**答**：

| 场景 | 推荐方案 |
|------|----------|
| 纯缓存（数据可从 DB 重建） | 关闭持久化 |
| 一般业务 | 混合持久化 + everysec |
| 不能丢数据 | AOF + always |

详见 [[66.归档发布/03.缓存/01-Redis持久化机制\|01-Redis持久化机制]]。

## 8. Redis 事务

**问**：Redis 支持事务吗？

**答**：支持，但和数据库事务不同。

```bash
MULTI          # 开始事务
SET key1 value1
SET key2 value2
EXEC           # 执行事务
```

Redis 事务的特点：
- **不支持回滚**：命令语法错误会报错，但已执行的命令不会回滚
- **不保证原子性**：某条命令失败，其他命令继续执行
- **不支持隔离**：事务执行期间，其他客户端可以插入命令

可以用 Lua 脚本实现更复杂的原子操作。

## 9. 大 Key 怎么处理

**问**：Redis 里有个 100MB 的 key，怎么删除？

**答**：用 `unlink` 异步删除，不要用 `del`。

```bash
UNLINK bigkey   # 异步删除，不阻塞
```

对于大 Hash、List，可以用渐进式删除：

```bash
# 删除大 Hash
HSCAN bigkey 0 COUNT 1000
HDEL bigkey field1 field2 ...

# 删除大 List
LTRIM bigkey 1000 -1   # 保留后部分，删除前 1000 个
```

## 10. Pipeline 和事务的区别

| | Pipeline | 事务 |
|--|----------|------|
| 原子性 | 不保证 | 不保证（但连续执行） |
| 返回值 | 所有命令执行完一次性返回 | 每条命令立即返回 |
| 网络开销 | 减少 RTT | 减少 RTT |
| 使用场景 | 批量执行不相关命令 | 需要连续执行的命令 |

```java
// Pipeline 示例
Pipeline pipelined = jedis.pipelined();
for (int i = 0; i < 1000; i++) {
    pipelined.set("key:" + i, "value:" + i);
}
pipelined.sync();
```

## 11. Redis 如何实现分布式锁

**答**：

```bash
# 加锁：SET key value NX EX 30
SET lock:user:1001 "uuid" NX EX 30

# 释放锁：Lua 脚本保证原子性
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

关键点：
- **NX**：只在 key 不存在时设置
- **EX**：设置过期时间，防止死锁
- **value 用 UUID**：保证只能释放自己的锁
- **Lua 脚本释放**：判断和删除要原子执行

生产环境建议用 Redisson，自带 Watchdog 自动续期。

## 12. 如何实现排行榜

**答**：用 ZSet，score 存分数。

```bash
# 添加用户分数
ZADD leaderboard 100 "user:1"
ZADD leaderboard 85 "user:2"
ZADD leaderboard 95 "user:3"

# 获取排行榜前 10
ZREVRANGE leaderboard 0 9 WITHSCORES

# 获取用户排名（从 0 开始）
ZREVRANK leaderboard "user:1"

# 获取用户分数
ZSCORE leaderboard "user:1"
```

## 13. 如何实现消息队列

**答**：可以用 List 或 Stream。

**List 方式**（简单但功能有限）：
```bash
LPUSH queue "message"      # 生产者
BRPOP queue 0              # 消费者（阻塞等待）
```

**Stream 方式**（推荐，支持消费组）：
```bash
XADD mystream * field1 value1    # 生产者
XREADGROUP GROUP mygroup consumer1 COUNT 1 BLOCK 0 STREAMS mystream >   # 消费者
```

## 14. Redis 集群方案

| 方案 | 特点 | 适用场景 |
|------|------|---------|
| 主从复制 | 读写分离，无自动故障转移 | 小规模 |
| 哨兵模式 | 自动故障转移，无分片 | 中等规模 |
| Cluster | 自动故障转移 + 数据分片 | 大规模 |

详见 [[66.归档发布/03.缓存/02-Redis高可用与复制\|02-Redis高可用与复制]] 和 [[66.归档发布/03.缓存/03-Redis集群与分片\|03-Redis集群与分片]]。

## 15. 常见命令

```bash
# 查看内存使用
INFO memory

# 查看连接数
INFO clients

# 查看命令统计
INFO commandstats

# 查看慢查询
SLOWLOG GET 10

# 查看 key 过期时间
TTL key

# 查看 key 类型
TYPE key

# 查看 key 大小
MEMORY USAGE key
```


