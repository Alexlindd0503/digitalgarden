---
{"dg-publish":true,"permalink":"/66.归档发布/04.并发/CLH锁队列原理/","dg-note-properties":{"时间":"2026-03-21"}}
---

#java #并发 #aqs #锁

```ad-summary
title: 总结

- CLH 是一种基于单向链表的自旋锁，每个线程只在自己的前驱节点上自旋，避免了全局变量的竞争
- AQS 使用的是 CLH 的变种：把自旋改成阻塞（park），把单向链表改成双向链表
- 双向链表是为了支持节点取消时 O(1) 找到前驱，单向 CLH 做不到这点
- 理解 CLH 是理解 AQS 等待队列设计的基础
```

## 1. CLH 是什么？

CLH（Craig, Landin, and Hagersten）是三位作者名字的缩写，他们在 1993 年提出了这种基于链表的自旋锁算法。

核心思想：**每个等待锁的线程不在同一个全局变量上自旋，而是在自己前驱节点的状态变量上自旋**。这样每个线程只关注自己前面那个人有没有释放锁，避免了多个线程同时盯着一个变量导致的缓存行竞争（Cache Line Bouncing）。

## 2. 原始 CLH 的结构

每个线程持有一个节点，节点里只有一个 `locked` 状态变量：

```java
class CLHNode {
    volatile boolean locked = false;
}
```

加锁流程：

```java
// 伪代码
CLHNode myNode = new CLHNode();
myNode.locked = true;                    // 声明自己要抢锁
CLHNode predNode = tail.getAndSet(myNode); // 把自己接到队尾，拿到前驱
while (predNode.locked) {               // 在前驱的 locked 上自旋
    // 等待前驱释放
}
// 自旋结束，拿到锁
```

释放锁：

```java
myNode.locked = false; // 把自己的 locked 置为 false，后继线程自旋结束
```

队列示意：

```
tail → [Node-C locked=true] → [Node-B locked=true] → [Node-A locked=false（持锁）]
         ↑ 线程C在此自旋          ↑ 线程B在此自旋
```

## 3. CLH 的优势

**本地自旋，缓存友好**

传统自旋锁所有线程都盯着同一个全局变量，锁释放时所有等待线程的缓存行同时失效，产生大量无效的缓存同步流量（Cache Line Bouncing）。

CLH 每个线程只在自己前驱的变量上自旋，锁释放时只有一个线程的缓存行失效，扩展性好得多。

**公平性**

FIFO 顺序，先来先得，天然公平。

## 4. AQS 对 CLH 做了哪些改造？

原始 CLH 直接用在 Java 里有两个问题，AQS 针对性地做了改造：

### 4.1 自旋 → 阻塞（park）

原始 CLH 是纯自旋，线程一直占用 CPU 等待。Java 里等待时间可能很长（比如等 IO、等数据库），一直自旋会白白烧 CPU。

AQS 的改造：线程自旋几次拿不到锁就调用 `LockSupport.park()` 挂起，锁释放时调用 `LockSupport.unpark()` 唤醒。

### 4.2 单向链表 → 双向链表

原始 CLH 是单向链表，节点只有 `prev` 方向的引用（通过 tail 入队，每个节点持有前驱引用）。

AQS 加了 `next` 指针，变成双向链表，原因有两个：

1. **节点取消**：线程因中断或超时取消排队时，需要把自己从队列里摘掉。摘除操作需要修改前驱的 `next` 指针，有了 `prev` 可以 O(1) 找到前驱，否则只能从头遍历 O(n)。详见 [[66.归档发布/04.并发/京东面试题：为什么AQS锁使用双向链表？\|京东面试题：为什么AQS锁使用双向链表？]]

2. **唤醒后继**：锁释放时通过 `next` 直接找到后继节点唤醒，不需要遍历。

### 4.3 增加节点状态 waitStatus

原始 CLH 节点只有 `locked` 一个布尔值。AQS 扩展为 `waitStatus`，支持更丰富的状态：

| waitStatus | 值 | 含义 |
|---|---|---|
| CANCELLED | 1 | 线程已取消，节点需要被跳过 |
| SIGNAL | -1 | 后继节点需要被唤醒 |
| CONDITION | -2 | 节点在条件队列中等待 |
| PROPAGATE | -3 | 共享模式下需要向后传播唤醒 |
| 初始值 | 0 | 正常等待 |

## 5. CLH vs AQS 对比

| | 原始 CLH | AQS 变种 |
|---|---|---|
| 链表方向 | 单向（只有 prev） | 双向（prev + next） |
| 等待方式 | 自旋 | 自旋 + park 阻塞 |
| 节点状态 | locked（boolean） | waitStatus（int，多种状态） |
| 节点取消 | 不支持 | 支持（O(1) 摘除） |
| 适用场景 | 短临界区，低延迟 | 通用，支持超时/中断 |

[[66.归档发布/04.并发/AQS抽象队列同步器原理\|AQS]] 本质上是把 CLH 从"适合短自旋"改造成了"适合 Java 通用并发场景"的版本，在保留本地自旋、公平 FIFO 优点的同时，解决了自旋浪费 CPU 和无法取消的问题。
