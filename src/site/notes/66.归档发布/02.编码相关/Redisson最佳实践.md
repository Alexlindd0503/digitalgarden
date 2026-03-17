---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Redisson最佳实践/"}
---

#java #最佳实践 #redis

```ad-summary
title: 总结

- key 要有业务含义，针对具体资源加锁，别搞全局锁
- leaseTime 传 -1 启用 Watchdog 自动续期，不用担心业务没跑完锁就过期
- finally 里释放锁前必须判断 `isHeldByCurrentThread()`，否则可能解掉别的线程的锁
```

## 标准用法

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
