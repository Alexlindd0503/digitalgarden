---
{"dg-publish":true,"permalink":"/66.归档发布/04.并发/AQS抽象队列同步器原理/","dg-note-properties":{"时间":"2026-03-15"}}
---

#java #并发 #aqs

```ad-summary
title: 总结

- AQS 用一个 `state` 变量 + FIFO 等待队列实现同步，ReentrantLock/Semaphore/CountDownLatch 都基于它
- 默认用非公平锁，吞吐量更高；需要顺序性才用公平锁
- 没有特殊需求用 `synchronized`，需要超时/中断/公平性再上 ReentrantLock
```

## 1. AQS 是什么？

AQS（AbstractQueuedSynchronizer）是 Java 并发包的底层框架，ReentrantLock、Semaphore、CountDownLatch、CyclicBarrier、ReentrantReadWriteLock 都是基于它实现的。

核心思路：用一个 `int` 类型的 `state` 变量表示同步状态，CAS 保证修改原子性。抢锁成功就修改 state，失败就进 FIFO 等待队列挂起，锁释放时唤醒队头线程重新抢。

## 2. state 在不同同步器里的含义

| 同步器 | state 含义 |
|--------|-----------|
| ReentrantLock | 0 = 未锁定，> 0 = 重入次数 |
| Semaphore | 剩余可用许可数 |
| CountDownLatch | 还需等待的计数 |
| ReentrantReadWriteLock | 高 16 位 = 读锁次数，低 16 位 = 写锁次数 |

## 3. 公平锁 vs 非公平锁

公平锁：线程来了先看队列有没有人在等，有就乖乖排队，严格 FIFO。
非公平锁：线程来了直接 CAS 抢，抢到就用，抢不到再排队。

默认用非公平锁，吞吐量更高。公平锁线程切换频繁，性能明显低一截，除非业务明确要求顺序性才用。

## 4. 为什么用双向链表？

AQS 的等待队列是 [[66.归档发布/04.并发/CLH锁队列原理\|CLH 锁队列]]的变种，原始 CLH 是单向链表，AQS 改成了双向链表。原因是线程可能因为中断或超时被取消，取消时需要修改前驱节点的 `next` 指针，有了 `prev` 才能 O(1) 找到前驱完成摘除，单向链表只能从头遍历 O(n)。详见 [[66.归档发布/04.并发/京东面试题：为什么AQS锁使用双向链表？\|京东面试题：为什么AQS锁使用双向链表？]]。

## 5. 常用同步器用法

```java
// ReentrantLock，记得 finally 里释放
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();
}

// Semaphore，限制并发数
Semaphore semaphore = new Semaphore(3);
semaphore.acquire();
try {
    // 最多 3 个线程同时进来
} finally {
    semaphore.release();
}

// CountDownLatch，等待多个任务完成
CountDownLatch latch = new CountDownLatch(5);
// 每个子任务完成后
latch.countDown();
// 主线程等所有子任务完成
latch.await();

// ReentrantReadWriteLock，读多写少场景
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();   // 读锁，多个线程可以同时持有
rwLock.writeLock().lock();  // 写锁，独占
```

## 6. 和 synchronized 的区别

| | synchronized | ReentrantLock（AQS）|
|--|-------------|-------------------|
| 实现层面 | JVM 内置 | Java 代码 |
| 释放锁 | 自动 | 手动，必须 finally |
| 可中断 | 否 | 是（lockInterruptibly）|
| 超时获取 | 否 | 是（tryLock）|
| 公平锁 | 否 | 支持 |
| 性能 | JDK 6 后差距不大 | 高并发下略好 |

没有特殊需求用 `synchronized` 就够了，代码更简洁不容易出错。需要超时、中断、公平性这些特性时再上 ReentrantLock。

## 相关链接

- [[66.归档发布/04.并发/死锁问题与解决方案\|死锁问题与解决方案]]
- [[02.并发工具类\|02.并发工具类]] — Lock/Semaphore/ReadWriteLock 等工具类使用
- [[管程是解决并发问题的万能钥匙\|管程是解决并发问题的万能钥匙]] — AQS 背后的管程思想
