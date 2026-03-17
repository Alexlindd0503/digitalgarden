---
{"dg-publish":true,"permalink":"/66.归档发布/03.缓存/Redis常见问题与解决方案清单/"}
---

#redis #缓存 #最佳实践

```ad-summary
title: 总结

- 雪崩：大量 key 同时过期或 Redis 宕机，过期时间加随机偏移 + 多级缓存
- 击穿：热点 key 过期，互斥锁或逻辑过期解决
- 穿透：查不存在的数据，布隆过滤器拦截或缓存空值
- 内存碎片率 > 1.5 开启 `activedefrag`；`evicted_keys` 涨了检查 bigkey
```

## 1. 缓存三大问题

### 1.1 缓存雪崩

大量 key 同时过期，或 Redis 宕机，请求全部打到数据库。

解决：
- 过期时间加随机偏移，避免集中失效：`EX (3600 + random(300))`
- 热点数据永不过期
- 多级缓存（本地缓存 + Redis + DB）
- 限流降级保护数据库

### 1.2 缓存击穿

热点 key 过期，瞬间大量并发请求打到数据库。

解决：
- 热点数据永不过期
- 互斥锁：只让一个请求去查 DB，其他等待缓存重建
- 逻辑过期：value 里存过期时间，异步更新，不真正让 key 失效

### 1.3 缓存穿透

查询根本不存在的数据，缓存和 DB 都没有，每次都透传到 DB。

解决：
- 布隆过滤器拦截（见下节）
- 缓存空值，设短过期时间（60s）
- 入参校验，非法请求直接拒绝

缓存一致性问题（延迟双删、分布式锁）见 [[66.归档发布/03.缓存/缓存一致性\|缓存一致性]]。

## 2. 布隆过滤器

用一个 bit 数组 + K 个哈希函数判断元素是否存在：
- 判断**不存在**：一定不存在
- 判断**存在**：可能存在（有误判率）

不支持删除，因为删一个 bit 会影响其他元素。需要删除场景可以用计数布隆过滤器，或定期重建。

误判率由数组长度（m）、哈希函数数量（k）、元素数量（n）决定，m 越大误判率越低。

```java
// Guava 布隆过滤器示例
BloomFilter<String> filter = BloomFilter.create(
    Funnels.stringFunnel(Charset.defaultCharset()),
    1000000,  // 预期元素数量
    0.01      // 误判率 1%
);
filter.put("key1");
filter.mightContain("key1"); // true（可能存在）
filter.mightContain("key2"); // false（一定不存在）
```

## 3. 数据倾斜

**数据量倾斜**：某个节点数据特别多，通常是 bigkey 导致。
- 拆分 bigkey（大 Hash 拆成多个小 Hash，大 List 分页存储）
- 使用 Cluster 自动均衡分片

**访问倾斜**：某个节点访问特别频繁，热点 key 集中。
- 热点数据复制到多个节点，key 加后缀分散：`hot_key_1`、`hot_key_2`
- 本地缓存热点数据，减少 Redis 访问

## 4. 内存问题

### 4.1 内存碎片

```bash
INFO memory
# mem_fragmentation_ratio > 1.5 需要处理
```

开启自动碎片整理：

```conf
config set activedefrag yes
active-defrag-ignore-bytes 100mb   # 碎片超过 100MB 开始清理
active-defrag-threshold-lower 10   # 碎片率超过 10% 开始清理
```

### 4.2 内存淘汰策略

```conf
maxmemory 2gb
maxmemory-policy allkeys-lru  # 推荐，淘汰最近最少使用的 key
```

常用策略：
- `allkeys-lru`：通用场景首选
- `volatile-lru`：只淘汰有过期时间的 key
- `noeviction`：不淘汰，内存满了直接报错（默认，生产别用）

### 4.3 缓冲区溢出

客户端输出缓冲区满会导致连接断开，bigkey 是主要原因。

```conf
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

## 相关链接

- [[66.归档发布/03.缓存/缓存一致性\|缓存一致性]]
- [[66.归档发布/03.缓存/缓存一致性解决方案\|缓存一致性解决方案]]
