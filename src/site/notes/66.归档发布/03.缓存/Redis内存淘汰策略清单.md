---
{"dg-publish":true,"permalink":"/66.归档发布/03.缓存/Redis内存淘汰策略清单/"}
---

#redis #缓存 #最佳实践

```ad-summary
title: 总结

- 内存满了会按 `maxmemory-policy` 淘汰 key，默认 `noeviction` 直接报错，生产别用
- 大部分场景用 `allkeys-lru`；有数据不能被淘汰时用 `volatile-lru`
- `evicted_keys` 一直涨说明内存不够用，要么加内存要么排查 bigkey
```

## 1. 内存淘汰是什么？

Redis 内存达到 `maxmemory` 上限时，会根据 `maxmemory-policy` 决定怎么处理新的写入。没配好的话，轻则写入报错，重则把热点数据给淘汰掉。

基本配置：

```conf
maxmemory 4gb
maxmemory-policy allkeys-lru
maxmemory-samples 5  # LRU/LFU 采样数，越大越精准但性能略低
```

`maxmemory` 建议设为物理内存的 60%~70%，给系统留点余量。

## 2. 八种策略对比

| 策略 | 淘汰范围 | 算法 | 适用场景 |
|------|---------|------|---------|
| `noeviction` | 不淘汰 | - | 数据不能丢，宁可报错 |
| `allkeys-lru` | 所有 key | LRU | 通用缓存，首选 |
| `volatile-lru` | 有过期时间的 key | LRU | 部分数据需要永久保留 |
| `allkeys-lfu` | 所有 key | LFU | 需要按访问频率淘汰（4.0+）|
| `volatile-lfu` | 有过期时间的 key | LFU | 频率淘汰 + 部分永久（4.0+）|
| `allkeys-random` | 所有 key | 随机 | 无冷热数据区分 |
| `volatile-random` | 有过期时间的 key | 随机 | 无冷热 + 部分永久 |
| `volatile-ttl` | 有过期时间的 key | TTL 最短优先 | 优先淘汰快过期的数据 |

`noeviction` 是默认值，生产环境基本不用，内存满了直接返回 OOM 错误。

## 3. 怎么选？

大部分场景直接用 `allkeys-lru`，有明显冷热数据时命中率最好。

有些数据不能被淘汰（比如配置、用户 session），就用 `volatile-lru`，重要数据不设过期时间，普通缓存设过期时间：

```java
// 重要数据，不设过期时间，不会被淘汰
redisTemplate.opsForValue().set("config:key", value);

// 普通缓存，设过期时间，内存紧张时可以淘汰
redisTemplate.opsForValue().set("cache:key", value, 30, TimeUnit.MINUTES);
```

访问模式比较均匀、没有明显热点，可以考虑 `allkeys-lfu`，它按访问频率淘汰，比 LRU 更精准，但性能略低一点。

## 4. LRU vs LFU

LRU 只看最后一次访问时间，LFU 看整个生命周期的访问次数。

Redis 的 LRU 是近似实现，不维护完整链表，而是随机采样 N 个 key（默认 5 个），从里面挑最久未访问的淘汰。性能好，但不是 100% 精确。

LFU 的问题是历史访问频率高的数据很难被淘汰，即使它已经很久没人用了。Redis 用了衰减机制来解决这个问题，随时间推移频率会慢慢降低。

## 5. 监控淘汰情况

```bash
INFO memory
# evicted_keys：被淘汰的 key 总数，这个数一直在涨说明内存不够用了
# used_memory / maxmemory：当前使用比例
```

`evicted_keys` 频繁增长，要么加内存，要么检查是不是有 bigkey 或者缓存设计有问题。

## 相关链接

- [[66.归档发布/03.缓存/Redis常见问题与解决方案清单\|Redis常见问题与解决方案清单]]
- [[66.归档发布/03.缓存/缓存一致性\|缓存一致性]]
