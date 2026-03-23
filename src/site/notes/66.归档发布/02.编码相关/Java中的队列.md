---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Java中的队列/","dg-note-properties":{"时间":"2026-03-21"}}
---

#java #并发 #最佳实践

```ad-summary
title: 总结

- 并发队列分两类：阻塞队列（基于锁）和非阻塞队列（基于 CAS）
- 阻塞队列实现了 BlockingQueue 接口，线程安全，队满/队空时自动阻塞
- 生产环境必须用有界队列，无界队列（如 LinkedBlockingQueue 默认）会 OOM
- 选型核心：有界用 ArrayBlockingQueue，高吞吐用 LinkedBlockingQueue，延迟任务用 DelayQueue，无锁高性能用 ConcurrentLinkedQueue
```

## 1. 整体分类

![Java并发队列](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/Java并发队列.png)

JDK 并发队列按实现方式分为两大类：

| 类型 | 实现机制 | 特点 |
|---|---|---|
| 阻塞队列 | ReentrantLock + Condition | 队满/队空时线程阻塞等待，线程安全 |
| 非阻塞队列 | CAS 原子操作 | 不阻塞线程，并发性能更高 |

阻塞队列底层依赖 [[66.归档发布/04.并发/AQS抽象队列同步器原理\|AQS]] 提供的 `ReentrantLock` 和 `Condition` 实现等待/唤醒机制；非阻塞队列则依赖 [[66.归档发布/04.并发/CAS学习\|CAS]] 保证原子性。

## 2. 阻塞队列

### ArrayBlockingQueue

有界数组队列，最常用。初始化必须指定容量。

内部用一把 `ReentrantLock` + `notEmpty`、`notFull` 两个 `Condition` 控制并发：
- 队列为空时，消费者在 `notEmpty` 上等待
- 队列已满时，生产者在 `notFull` 上等待

读写共用同一把锁，生产和消费互斥。

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(100);
queue.put("task");       // 队满则阻塞
queue.offer("task", 1, TimeUnit.SECONDS); // 超时返回 false
String task = queue.take(); // 队空则阻塞
```

### LinkedBlockingQueue

链表实现，可有界可无界（默认 `Integer.MAX_VALUE`，生产环境务必指定容量）。

用 `takeLock` 和 `putLock` 两把独立的锁，读写不互斥，吞吐量高于 `ArrayBlockingQueue`。

```java
// 生产环境必须指定容量，否则无界队列可能 OOM
BlockingQueue<String> queue = new LinkedBlockingQueue<>(1000);
```

[[66.归档发布/04.并发/线程池ThreadPoolExecutor\|线程池]] 的 `newFixedThreadPool` 和 `newSingleThreadExecutor` 默认使用的就是无界 `LinkedBlockingQueue`，这也是为什么生产环境不推荐直接用 `Executors` 工厂方法的原因之一。

### PriorityBlockingQueue

基于最小堆的无界优先级队列，每次出队返回优先级最高的元素。

元素必须实现 `Comparable` 接口或传入 `Comparator`。因为是无界队列，`put` 不会阻塞，只有 `take` 在队空时阻塞。扩容时先释放锁再用 CAS 保证只有一个线程执行扩容。

```java
PriorityBlockingQueue<Task> queue = new PriorityBlockingQueue<>();
// Task 实现 Comparable，优先级高的先出队
```

### DelayQueue

支持延迟获取的无界阻塞队列，底层用 `PriorityQueue` 存储。

元素必须实现 `Delayed` 接口，只有延迟时间到期的元素才能被取出。常用于：
- 缓存过期清理
- 定时任务调度
- 订单超时取消

```java
public class DelayTask implements Delayed {
    private final long expireTime; // 到期时间戳

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(expireTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.expireTime, ((DelayTask) o).expireTime);
    }
}

DelayQueue<DelayTask> queue = new DelayQueue<>();
queue.put(new DelayTask(5000)); // 5 秒后才能取出
DelayTask task = queue.take();  // 阻塞直到有到期元素
```

### SynchronousQueue

无缓冲队列，内部不存储任何元素。每一个 `put` 必须等待一个 `take`，反之亦然，生产者和消费者必须配对才能完成传递。

[[66.归档发布/04.并发/线程池ThreadPoolExecutor\|线程池]] 的 `newCachedThreadPool` 使用的就是 `SynchronousQueue`：有空闲线程就直接交付任务，没有就新建线程，实现了任务的即时传递。

```java
SynchronousQueue<String> queue = new SynchronousQueue<>();
// 生产者线程
new Thread(() -> queue.put("data")).start();
// 消费者线程，两者必须同时就绪
new Thread(() -> queue.take()).start();
```

### LinkedTransferQueue

无界阻塞队列，可以看作 `LinkedBlockingQueue` + `SynchronousQueue` 的结合体。

`transfer()` 方法：如果有等待的消费者，直接交付；否则入队等待消费者取走。相比 `SynchronousQueue` 多了内部缓冲，相比 `LinkedBlockingQueue` 用 CAS 无锁操作性能更好。

## 3. 非阻塞队列

### ConcurrentLinkedQueue

基于单向链表的无界非阻塞队列，用 CAS 保证线程安全，不阻塞线程。

适合读多写少、对延迟敏感的场景。`size()` 方法需要遍历链表，时间复杂度 O(n)，高频调用要注意性能。

```java
ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();
queue.offer("item");
String item = queue.poll(); // 队空返回 null，不阻塞
```

### ConcurrentLinkedDeque

双端队列，支持从头部和尾部同时插入/删除，同样基于 CAS 实现。

适合多生产者多消费者、需要双端操作的场景（如工作窃取算法）。

## 4. BlockingQueue 核心 API

![BlockingQueue](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/BlockingQueue.png)

![方法](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/方法.png)

| 方法 | 队满/队空时的行为 | 说明 |
|---|---|---|
| `add(e)` | 抛出异常 | 队满抛 `IllegalStateException` |
| `offer(e)` | 返回 false | 不阻塞 |
| `put(e)` | 阻塞等待 | 直到有空位 |
| `offer(e, t, unit)` | 超时返回 false | 等待指定时间 |
| `remove()` | 抛出异常 | 队空抛 `NoSuchElementException` |
| `poll()` | 返回 null | 不阻塞 |
| `take()` | 阻塞等待 | 直到有元素 |
| `poll(t, unit)` | 超时返回 null | 等待指定时间 |

生产环境推荐用 `offer` + 超时 或 `put`，避免用 `add` / `remove` 抛异常的方式。

## 5. 选型指南

| 场景 | 推荐队列 |
|---|---|
| 通用有界缓冲，读写均衡 | `ArrayBlockingQueue` |
| 高吞吐生产消费，读写分离 | `LinkedBlockingQueue`（指定容量） |
| 按优先级处理任务 | `PriorityBlockingQueue` |
| 延迟/定时任务 | `DelayQueue` |
| 线程间直接交付，无需缓冲 | `SynchronousQueue` |
| 无锁高性能，不需要阻塞 | `ConcurrentLinkedQueue` |
| 需要双端操作 | `ConcurrentLinkedDeque` |
