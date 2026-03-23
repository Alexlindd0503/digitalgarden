---
{"dg-publish":true,"permalink":"/66.归档发布/03.缓存/06-Redis开发规范与最佳实践/","dg-note-properties":{"时间":"2026-03-23"}}
---

#redis #规范 #最佳实践 #Redisson

```ad-summary
title: 这篇笔记讲什么

- 强制规范：禁用 KEYS、FLUSHALL、FLUSHDB
- Key 设计：业务前缀 + 缩写，控制长度
- Value 设计：高效序列化，控制大小
- Redisson 最佳实践：Watchdog 自动续期，finally 释放锁
```

## 1. 强制规范

这几条是红线，必须遵守。搞不好就是生产事故，跟[[66.归档发布/03.缓存/04-Redis缓存问题与解决方案\|缓存问题]]一样严重。

| 规范 | 原因 |
|------|------|
| 禁用 `KEYS *` | 全量遍历，会阻塞主线程 |
| 禁用 `FLUSHALL` | 清空所有数据，生产事故 |
| 禁用 `FLUSHDB` | 清空当前库，生产事故 |

可以在配置文件里用 `rename-command` 把这些命令改名或禁用：

```bash
rename-command KEYS ""
rename-command FLUSHALL ""
```

## 2. Key 设计规范

### 2.1 用业务前缀

格式：`业务名:模块名:具体标识`

```bash
# 好的
user:profile:1001
order:detail:20240323001

# 不好的
u1001
order1
```

用缩写控制长度，Key 越短越省内存。

### 2.2 控制 Key 长度

Key 太长会占用内存，建议不超过 128 字节。

## 3. Value 设计规范

### 3.1 String 类型不超过 10KB

Value 太大会影响网络传输和内存占用。

### 3.2 集合类型不超过 1 万个元素

Hash、List、Set、SortedSet 的元素太多，操作会变慢。

### 3.3 用高效的序列化方式

| 方式 | 特点 |
|------|------|
| JSON | 可读性好，但体积大 |
| Protobuf | 体积小，但不可读 |
| MessagePack | 体积小，速度快 |

## 4. 实例容量规范

| 规范 | 原因 |
|------|------|
| 不同业务分不同实例 | 避免互相影响，方便扩容 |
| 实例容量 2~6GB | 太大 fork 慢，主从同步耗时长 |
| 只存热数据 | 冷数据放数据库，省内存 |

## 5. 命令使用规范

| 命令 | 建议 |
|------|------|
| `MONITOR` | 慎用，会输出所有命令，性能损耗大 |
| 全量操作（`HGETALL`、`SMEMBERS`） | 慎用，数据量大时阻塞 |
| 过期时间 | 写入时必须设置，避免内存泄漏 |
| 整数共享池 | 0~9999 的整数会复用对象，省内存 |

## 6. Redisson 最佳实践

### 6.1 标准用法

```java
String lockKey = "business_action:" + relevantId;
RLock lock = redissonClient.getLock(lockKey);
try {
    // waitTime: 等待时间，建议 3-5 秒
    // leaseTime 不传或传 -1，启用 Watchdog 自动续期
    boolean getLock = lock.tryLock(3, TimeUnit.SECONDS);
    if (!getLock) {
        log.warn("获取锁失败，key: {}", lockKey);
        throw new BizException("系统繁忙，请稍后重试");
    }
    // 执行业务逻辑
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    log.error("获取锁被中断", e);
    throw new BizException("操作中断");
} catch (Exception e) {
    log.error("业务执行异常", e);
    throw e;
} finally {
    // 必须判断是否是当前线程持有，否则会解掉别人的锁
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### 6.2 关键点

| 要点 | 说明 |
|------|------|
| key 要有业务含义 | 针对具体资源加锁，别搞全局锁 |
| leaseTime 传 -1 | 启用 Watchdog 自动续期（默认 30 秒） |
| finally 释放锁前判断 | 必须 `isHeldByCurrentThread()`，否则可能解掉别人的锁 |
| waitTime 3-5 秒 | 等待时间别太长，避免请求堆积 |

### 6.3 Watchdog 机制

Redisson 的 Watchdog 会自动给锁续期：
- 默认锁超时 30 秒
- Watchdog 每 10 秒检查一次，如果还持有锁就续期 30 秒
- 业务执行完释放锁后，Watchdog 停止

如果指定了 leaseTime（不是 -1），Watchdog 不会启动，锁到期自动释放。

## 7. 避免 Bigkey

Bigkey 会导致：
- 操作耗时长，阻塞主线程
- 网络传输慢
- 内存分布不均

| 类型 | 建议大小 |
|------|----------|
| String | < 10KB |
| Hash | < 5000 字段 |
| List | < 5000 元素 |
| Set | < 5000 元素 |
| ZSet | < 5000 元素 |

用 `unlink` 替代 `del` 删除大 key，异步删除不阻塞主线程。


