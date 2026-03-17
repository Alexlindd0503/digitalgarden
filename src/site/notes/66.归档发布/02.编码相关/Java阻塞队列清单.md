---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Java阻塞队列清单/"}
---

#java #并发 #最佳实践

```ad-summary
title: 总结

- 阻塞队列在队列空/满时自动阻塞，是生产者-消费者模式的标配
- LinkedBlockingQueue 读写分离锁，吞吐量高，线程池默认用它；注意要指定容量，默认无界容易 OOM
- DelayQueue 做延迟任务，SynchronousQueue 做直接交接，LinkedTransferQueue 性能最高
- 生产环境用带超时的方法（offer/poll），避免线程永久阻塞
```

阻塞队列在队列为空时 `take()` 会阻塞，队列满时 `put()` 会阻塞，是生产者-消费者模式的标配。

## 1. 六种实现

### 1.1 ArrayBlockingQueue

数组实现的有界队列，读写共用一把锁，并发性能一般。需要严格控制内存时用。

```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);
queue.put(task);   // 满了就阻塞
Task t = queue.take(); // 空了就阻塞
```

### 1.2 LinkedBlockingQueue

链表实现，读写分离锁（takeLock / putLock），生产者和消费者可以并行，吞吐量比 ArrayBlockingQueue 高。`Executors.newFixedThreadPool` 用的就是它。

```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000); // 建议指定容量，默认 Integer.MAX_VALUE 容易 OOM
```

### 1.3 PriorityBlockingQueue

无界优先级队列，底层最小堆，每次出队返回优先级最高的元素。元素需要实现 `Comparable`。

```java
BlockingQueue<PriorityTask> queue = new PriorityBlockingQueue<>();

class PriorityTask implements Comparable<PriorityTask> {
    int priority;
    @Override
    public int compareTo(PriorityTask o) {
        return Integer.compare(this.priority, o.priority);
    }
}
```

### 1.4 DelayQueue

延迟队列，元素到期才能取出。适合订单超时、缓存过期清理等场景。

```java
BlockingQueue<DelayedTask> queue = new DelayQueue<>();

class DelayedTask implements Delayed {
    private long executeTime;

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(executeTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.getDelay(TimeUnit.MILLISECONDS), o.getDelay(TimeUnit.MILLISECONDS));
    }
}
```

### 1.5 SynchronousQueue

容量为 0，每个 `put` 必须等一个 `take` 来配对，数据直接从生产者交给消费者。`Executors.newCachedThreadPool` 用的就是它，没有空闲线程就创建新线程。

```java
BlockingQueue<Task> queue = new SynchronousQueue<>();
```

### 1.6 LinkedTransferQueue

CAS 无锁实现，性能比 LinkedBlockingQueue 更高。有消费者等待时直接交接，没有则放入队列。

```java
TransferQueue<Task> queue = new LinkedTransferQueue<>();
queue.transfer(task);            // 等待消费者接收
queue.tryTransfer(task);         // 没有消费者立即返回 false
```

## 2. 怎么选

| 需求 | 推荐 |
|------|------|
| 控制内存，固定大小 | ArrayBlockingQueue |
| 高吞吐生产者-消费者 | LinkedBlockingQueue |
| 按优先级处理任务 | PriorityBlockingQueue |
| 延迟/定时执行 | DelayQueue |
| 生产消费速度匹配，不想堆积 | SynchronousQueue |
| 极致性能 | LinkedTransferQueue |

## 3. 常用方法

| 操作 | 阻塞 | 超时 | 不阻塞 |
|------|------|------|--------|
| 插入 | `put(e)` | `offer(e, time, unit)` | `offer(e)` |
| 取出 | `take()` | `poll(time, unit)` | `poll()` |

生产环境推荐用超时方法，避免线程永久阻塞。

## 4. 和非阻塞队列对比

| | 阻塞队列 | 非阻塞队列 |
|--|---------|-----------|
| 实现 | ReentrantLock | CAS 无锁 |
| 队列空/满时 | 阻塞等待 | 返回 null/false |
| 性能 | 中等 | 更高 |
| 编程复杂度 | 简单 | 需处理 null |
| 典型实现 | LinkedBlockingQueue | ConcurrentLinkedQueue |

用阻塞队列：需要自动等待、生产者-消费者模式、线程池任务队列。
用非阻塞队列：高并发低延迟、不希望线程阻塞、能处理 null 返回值。

## 相关链接

- [[66.归档发布/02.编码相关/Java非阻塞队列清单\|Java非阻塞队列清单]]
- [[66.归档发布/04.并发/线程池ThreadPoolExecutor\|线程池ThreadPoolExecutor]]
