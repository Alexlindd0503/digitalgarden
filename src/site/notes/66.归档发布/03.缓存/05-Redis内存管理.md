---
{"dg-publish":true,"permalink":"/66.归档发布/03.缓存/05-Redis内存管理/","dg-note-properties":{"时间":"2026-03-23"}}
---

#redis #内存管理 #淘汰策略 #内存碎片

```ad-summary
title: 这篇笔记讲什么

- 8 种淘汰策略：大部分场景用 allkeys-lru
- LRU vs LFU：LRU 看最近访问时间，LFU 看访问频率
- 内存碎片率 > 1.5 开启 activedefrag
- evicted_keys 一直涨说明内存不够用
```

## 1. 内存淘汰策略

Redis 内存达到 `maxmemory` 上限时，会根据 `maxmemory-policy` 决定怎么处理新的写入。这个跟[[66.归档发布/03.缓存/04-Redis缓存问题与解决方案\|缓存三大问题]]有关联，内存满了不处理好，雪崩击穿都来了。

### 1.1 基本配置

```conf
maxmemory 4gb
maxmemory-policy allkeys-lru
maxmemory-samples 5  # LRU/LFU 采样数，越大越精准但性能略低
```

`maxmemory` 建议设为物理内存的 60%~70%，给系统留点余量。

### 1.2 八种策略

| 策略 | 淘汰范围 | 算法 | 适用场景 |
|------|---------|------|---------|
| `noeviction` | 不淘汰 | - | 数据不能丢，宁可报错 |
| `allkeys-lru` | 所有 key | LRU | 通用缓存，首选 |
| `volatile-lru` | 有过期时间的 key | LRU | 部分数据需要永久保留 |
| `allkeys-lfu` | 所有 key | LFU | 按访问频率淘汰（4.0+）|
| `volatile-lfu` | 有过期时间的 key | LFU | 频率淘汰 + 部分永久（4.0+）|
| `allkeys-random` | 所有 key | 随机 | 无冷热数据区分 |
| `volatile-random` | 有过期时间的 key | 随机 | 无冷热 + 部分永久 |
| `volatile-ttl` | 有过期时间的 key | TTL 最短优先 | 优先淘汰快过期的数据 |

`noeviction` 是默认值，生产环境基本不用，内存满了直接返回 OOM 错误。

### 1.3 怎么选

大部分场景直接用 `allkeys-lru`，有明显冷热数据时命中率最好。

有些数据不能被淘汰（比如配置、用户 session），就用 `volatile-lru`：

```java
// 重要数据，不设过期时间，不会被淘汰
redisTemplate.opsForValue().set("config:key", value);

// 普通缓存，设过期时间，内存紧张时可以淘汰
redisTemplate.opsForValue().set("cache:key", value, 30, TimeUnit.MINUTES);
```

### 1.4 LRU vs LFU

**LRU（Least Recently Used）**：最近最少使用。看的是"最后一次访问时间"，很久没访问的优先淘汰。

问题：某个 key 之前访问很频繁，突然不访问了，但因为最近访问过，不会被淘汰。

**LFU（Least Frequently Used）**：最不经常使用。看的是"访问频率"，访问次数少的优先淘汰。

Redis 的 LRU 是近似实现，随机采样 N 个 key（默认 5 个），从里面挑最久未访问的淘汰。

LFU 用了衰减机制，随时间推移频率会慢慢降低，避免历史热点数据一直占内存。

## 2. 内存碎片

Redis 申请和释放内存时，操作系统分配的内存块之间会产生一些用不上的小空隙，这就是内存碎片。

### 2.1 怎么查

用 `info memory` 命令，看 `mem_fragmentation_ratio`：

```bash
info memory
```

| 碎片率 | 状态 |
|--------|------|
| < 1 | 数据内存 > 系统分配内存，说明用了 swap，要警惕 |
| 1 ~ 1.5 | 正常范围 |
| > 1.5 | 碎片太多，需要清理 |

### 2.2 怎么清理

开启自动清理：

```bash
config set activedefrag yes
```

触发条件（必须同时满足）：

| 参数 | 含义 | 默认值 |
|------|------|--------|
| active-defrag-ignore-bytes | 碎片字节数达到这个值才清理 | 100MB |
| active-defrag-threshold-lower | 碎片占比达到这个百分比才清理 | 10% |

CPU 占比控制：

| 参数 | 含义 | 建议值 |
|------|------|--------|
| active-defrag-cycle-min | 最低 CPU 占比，保证清理能开展 | 25 |
| active-defrag-cycle-max | 最高 CPU 占比，超过就停止 | 75 |

**如果线上请求压力大，建议调小 cycle-max**，避免清理占用太多 CPU 影响正常请求。

## 3. 监控淘汰情况

```bash
INFO memory
# evicted_keys：被淘汰的 key 总数，这个数一直在涨说明内存不够用了
# used_memory / maxmemory：当前使用比例
```

`evicted_keys` 频繁增长，要么加内存，要么检查是不是有 bigkey 或者缓存设计有问题。


