---
{"dg-publish":true,"permalink":"/66.归档发布/04.并发/线程池ThreadPoolExecutor/"}
---

#面试 #java
## 1. 核心参数有哪些？
- **corePoolSize**：核心线程数
- **maxCorePoolSize**：核心线程不够用时，能扩容到的最大线程数
- **keepAliveTime**：扩容出来的线程最多闲置多久，到期就销毁
- **queue**：任务队列
- **RejectedExectionHandler**：队列满了之后的拒绝策略
  - AbortPolicy：直接抛异常
  - DiscardPolicy：直接丢弃
  - DiscardOldestPolicy：丢弃最老的任务
  - CallerRunsPolicy：让调用线程自己执行
  - 自定义策略
- **ThreadFactory**：怎么创建新线程

---

## 2. 工作原理是什么？

1. **刚开始**：线程池里没有线程，来一个任务就创建一个 worker 线程去处理，直到达到 corePoolSize
2. **超过核心线程数**：新任务就扔到队列里。队列满了，就尝试继续创建线程（不超过 maxSize）
   - 小技巧：把 corePoolSize 和 maxCorePoolSize 设成一样，这样就优先创建线程，再用队列
3. **队列清空后**：如果线程数大于 coreSize，就开始清理多余的线程。判断 keepAliveTime 到了没，到了就销毁，始终保持 corePoolSize
4. **有界队列满了**：触发拒绝策略 RejectedExectionHandler
5. **无界队列**：内存会一直涨，最后可能 OOM

![jdk线程池原理.png](/img/user/attachment/jdk%E7%BA%BF%E7%A8%8B%E6%B1%A0%E5%8E%9F%E7%90%86.png)

---

## 3. 要注意什么？

### 3.1 JDK 线程池适合 CPU 密集型任务

JDK 的线程池会**优先把任务扔队列**，而不是创建更多线程。为什么这么设计？

因为 CPU 密集型任务（大量计算）会让 CPU 很忙，这时候线程数和 CPU 核数差不多就够了。线程太多反而会频繁切换上下文，降低效率。所以超过核心线程数后，线程池不会马上加线程，而是让任务在队列里等核心线程空闲。

Tomcat 使用的线程池就不是 JDK 原生的线程池，而是做了一些改造，当线程数超过 coreThreadCount 之后会优先创建线程，直到线程数到达maxThreadCount，这样就比较适合于 Web 系统大量 IO 操作的场景了。
### 3.2 队列堆积要监控
队列堆积量是个重要指标，特别是对实时性要求高的任务。堆积太多说明处理不过来了。

### 3.3 别用无界队列
用线程池一定记住：**别用无界队列**（没设大小限制的队列），不然内存会被撑爆。