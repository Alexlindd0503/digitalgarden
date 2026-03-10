---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Redisson最佳实践/"}
---


#java #最佳实践 #redis
```java
// 1. Key 应该具有业务意义，避免全局竞争
String lockKey = "business_action:" + relevantId; 
RLock lock = null;
try {
    // 2. 获取非公平锁（推荐），性能更好
    lock = redissonClient.getLock(lockKey);
    // 3. 尝试获取锁
    // 参数1: waitTime 等待时间 (建议根据业务容忍度设置，如 3-5秒)
    // 参数2: leaseTime 不传或传 -1，启用 Watchdog 机制
    boolean getLock = lock.tryLock(3, TimeUnit.SECONDS);
    if (!getLock) {
        // 获取锁失败，快速失败或降级处理
        log.warn("获取锁失败，key: {}", lockKey);
        // 建议抛出异常或返回特定错误码，而不是静默处理
        throw new BizException("系统繁忙，请稍后重试");
    }
    // 4. 执行业务逻辑
    // do business...
} catch (InterruptedException e) {
    // 处理线程中断异常
    Thread.currentThread().interrupt();
    log.error("获取锁被中断", e);
    throw new BizException("操作中断");
} catch (Exception e) {
    // 业务异常或其他运行时异常
    log.error("业务执行异常", e);
    // 必须抛出异常，让调用方知晓失败
    throw e; 
} finally {
    // 5. 安全释放锁
    // 只有当前线程持有锁时才释放
    if (lock != null && lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}

```

IsHeldByCurrentThread 这个需要判断，否则会出现解锁解的不是当前线程的
