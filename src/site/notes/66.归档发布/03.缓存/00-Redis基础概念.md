---
{"dg-publish":true,"permalink":"/66.归档发布/03.缓存/00-Redis基础概念/","dg-note-properties":{"时间":"2026-03-23"}}
---

#redis #入门

```ad-summary
title: 这篇笔记讲什么

- Redis 是单线程内存数据库，支持持久化、主从复制、集群
- 五种基本数据结构：String、List、Hash、Set、ZSet
- 适用场景：缓存、计数器、排行榜、分布式锁、消息队列
```

## 1. Redis 是什么

Redis（Remote Dictionary Server）是个开源的内存数据库，读写速度特别快，微秒级响应。它支持[[66.归档发布/03.缓存/01-Redis持久化机制\|持久化]]、[[66.归档发布/03.缓存/02-Redis高可用与复制\|主从复制]]，还能搞[[66.归档发布/03.缓存/03-Redis集群与分片\|集群分片]]。

### 1.1 核心特点

| 特点 | 说明 |
|------|------|
| 内存存储 | 读写速度快，微秒级响应 |
| 单线程模型 | 避免多线程竞争，6.0 引入多线程 IO |
| 持久化 | 支持 RDB、AOF、混合持久化 |
| 丰富数据结构 | 5 种基本类型 + 多种扩展类型 |
| 主从复制 | 支持读写分离、数据备份 |
| 集群分片 | 支持水平扩展，16384 个哈希槽 |

### 1.2 为什么是单线程

Redis 用单线程处理命令，原因是：

- **内存操作快**：大部分操作在内存完成，CPU 不是瓶颈
- **避免锁竞争**：单线程无需加锁，减少开销
- **IO 多路复用**：用 epoll 同时处理多个连接，非阻塞

单线程的缺点：一个慢查询会阻塞所有后续命令。

## 2. 数据结构

### 2.1 五种基本类型

| 类型 | 底层实现 | 典型场景 |
|------|----------|----------|
| String | SDS（简单动态字符串） | 缓存、计数器、分布式锁 |
| List | 压缩列表 / 双端链表 | 消息队列、最新列表 |
| Hash | 压缩列表 / 哈希表 | 对象存储、用户信息 |
| Set | 整数数组 / 哈希表 | 标签、共同好友 |
| ZSet | 压缩列表 / 跳表 | 排行榜、延迟队列 |

### 2.2 扩展类型

| 类型 | 说明 | 场景 |
|------|------|------|
| Bitmap | 位图，按位存储 | 签到、布隆过滤器 |
| HyperLogLog | 基数统计 | UV 统计（误差 0.81%） |
| Geo | 地理位置 | 附近的人、距离计算 |
| Stream | 消息队列 | 消息发布订阅 |

## 3. 适用场景

| 场景 | 说明 |
|------|------|
| 缓存 | 热点数据缓存，减轻 DB 压力 |
| 计数器 | 文章阅读量、点赞数，原子操作 |
| 排行榜 | 用 ZSet 实现，score 存分数 |
| 分布式锁 | SETNX + 过期时间，或 Redisson |
| 消息队列 | List 或 Stream 实现 |
| 会话存储 | 用户 Session、Token 存储 |

## 4. 基本命令

```bash
# String
SET key value [EX seconds]  # 设置值，可带过期时间
GET key                     # 获取值
INCR key                    # 自增 1
INCRBY key increment        # 自增指定值

# Hash
HSET key field value        # 设置字段
HGET key field              # 获取字段
HGETALL key                 # 获取所有字段

# List
LPUSH key value             # 左端插入
RPOP key                    # 右端弹出
LRANGE key start stop       # 范围查询

# Set
SADD key member             # 添加元素
SMEMBERS key                # 获取所有元素
SINTER key1 key2            # 交集

# ZSet
ZADD key score member       # 添加带分数的元素
ZRANGE key start stop       # 按分数排序查询
ZRANK key member            # 获取排名
```


