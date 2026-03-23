---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Java非阻塞队列清单/","dg-note-properties":{"时间":"2026-03-14"}}
---

#java #并发 #最佳实践

```ad-summary
title: 总结

- 非阻塞队列用 CAS 无锁实现，操作立即返回，不阻塞线程，性能比阻塞队列高
- ConcurrentLinkedQueue 是高并发场景首选；ConcurrentLinkedDeque 支持双端，ForkJoinPool 工作窃取用它
- 代价是需要自己处理 poll() 返回 null 的情况
```

非阻塞队列用 CAS 无锁  操作实现线程安全，操作立即返回，不会让线程进入等待状态。高并发低延迟场景比阻塞队列性能更好，代价是需要自己处理 `poll()` 返回 `null` 的情况。

## 1. ConcurrentLinkedQueue

无界单向链表，FIFO，高并发场景首选。

```java
Queue<Task> queue = new ConcurrentLinkedQueue<>();

queue.offer(task);       // 入队，不会阻塞
Task t = queue.poll();   // 出队，空队列返回 null
Task h = queue.peek();   // 查看队首，不移除
```

## 2. ConcurrentLinkedDeque

无界双向链表，支持从两端操作，ForkJoinPool 的工作窃取就用的这个。

```java
Deque<Task> deque = new ConcurrentLinkedDeque<>();

deque.offerFirst(task);       // 头部插入
deque.offerLast(task);        // 尾部插入
Task t = deque.pollFirst();   // 头部取出
Task t = deque.pollLast();    // 尾部取出（工作窃取从这里偷）
```

## 3. 和阻塞队列怎么选

|       | 非阻塞队列    | 阻塞队列          |
| ----- | -------- | ------------- |
| 实现    | CAS 无锁   | ReentrantLock |
| 队列空时  | 返回 null  | 阻塞等待          |
| 性能    | 更高       | 中等            |
| 编程复杂度 | 需处理 null | 简单            |

用非阻塞队列：高并发、低延迟、不希望线程阻塞。

用阻塞队列：生产者-消费者模式、希望自动等待、不想处理 null。

参考资料：
[[66.归档发布/04.并发/CAS学习\|CAS学习]]