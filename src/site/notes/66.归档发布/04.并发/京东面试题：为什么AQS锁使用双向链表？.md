---
{"dg-publish":true,"permalink":"/66.归档发布/04.并发/京东面试题：为什么AQS锁使用双向链表？/","dg-note-properties":{"时间":"2026-03-21"}}
---

#java #并发 #面试 #aqs

```ad-summary
title: 总结

- AQS 等待队列是 CLH 锁队列的变种，用双向链表实现 FIFO
- 用双向链表的核心原因：节点取消时需要 O(1) 找到前驱节点并摘除，单向链表做不到
- 锁释放时直接通过 `next` 唤醒后继节点，不需要遍历
- 相比数组、红黑树、跳表，双向链表更轻量，完全满足 FIFO 场景需求
```

## 1. AQS 的等待队列结构

[[66.归档发布/04.并发/AQS抽象队列同步器原理\|AQS]]（AbstractQueuedSynchronizer）是 Java 并发包的核心框架，`ReentrantLock`、`Semaphore`、`CountDownLatch` 等都基于它实现。

AQS 内部维护一个 FIFO 等待队列，底层是 **CLH 锁队列的变种**，节点结构如下：

```java
static class Node {
    volatile Node prev;    // 前驱节点
    volatile Node next;    // 后继节点
    volatile Thread thread; // 当前节点持有的线程
    volatile int waitStatus; // 节点状态（CANCELLED、SIGNAL 等）
    // ...
}
```

## 2. 为什么选双向链表？

### 2.1 节点取消需要 O(1) 找到前驱

这是最核心的原因。

线程在等待锁的过程中，可能因为**中断**或**超时**而取消排队。取消时需要把这个节点从队列里摘掉，操作是：

```
前驱节点.next = 当前节点.next
```

如果是单向链表，找前驱节点只能从头遍历，时间复杂度 O(n)。双向链表直接通过 `prev` 拿到前驱，O(1) 完成。

```java
// AQS 取消节点的核心逻辑（简化）
private void cancelAcquire(Node node) {
    Node pred = node.prev; // 直接拿到前驱，O(1)
    node.waitStatus = Node.CANCELLED;
    // 调整指针，跳过当前节点
    pred.next = node.next;
}
```

### 2.2 锁释放时快速唤醒后继节点

锁持有者释放锁时，需要唤醒队列中下一个等待的线程。直接通过 `next` 指针拿到后继节点，O(1) 完成唤醒，不需要遍历。

```java
// AQS 释放锁唤醒后继（简化）
private void unparkSuccessor(Node node) {
    Node next = node.next; // 直接拿后继，O(1)
    if (next != null && next.waitStatus <= 0) {
        LockSupport.unpark(next.thread);
    }
}
```

### 2.3 跳过已取消节点，保证公平性

队列中可能存在 `waitStatus = CANCELLED` 的已取消节点。唤醒时需要跳过这些节点，找到第一个有效的等待线程。

有了 `prev` 指针，可以从后往前扫描，快速定位到最近的有效前驱，保证公平唤醒顺序。

## 3. 为什么不用其他数据结构？

| 数据结构 | 问题 |
|---|---|
| 单向链表 | 节点取消时找前驱需要 O(n) 遍历 |
| 数组 | 大小固定，扩容开销大，不适合动态排队 |
| 红黑树 | 实现复杂，FIFO 场景用不上排序能力，杀鸡用牛刀 |
| 跳表 | 同上，复杂度高，FIFO 不需要随机访问 |

双向链表插入/删除 O(1)，结构简单，完全契合 AQS 的 FIFO 等待场景。
