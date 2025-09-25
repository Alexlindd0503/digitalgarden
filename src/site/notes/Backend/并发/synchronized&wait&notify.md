---
{"dg-publish":true,"permalink":"/Backend/并发/synchronized&wait&notify/"}
---

#面试 #锁 #原理 #线程通信

### 1. 理解 Monitor 监视器模型
在 JVM 中，每个对象都关联着一个监视器，如下图所示。
一个对象实例，拥有头对象信息，里面有 Mark Word，内部有个 monitor 指针，指向外部的 **ObjectMonitor 对象**（c++编写的），Class Metadata Address 指针，指向类信息对象。
![synchronized加锁原理-1.png](/img/user/Resource/%E9%99%84%E4%BB%B6/synchronized%E5%8A%A0%E9%94%81%E5%8E%9F%E7%90%86-1.png)

**ObjectMonitor 对象** 有两个两个关键的队列:
1. **入口队列 (Entry Set)**: 所有试图进入synchronized 同步代码块的线程，如果发现对象的锁已经被占用了，则进入这个队列进行等待。锁被释放的时候，他们会进行竞争。
2. **等待队列 (Wait Set)**: 当已经获得锁的线程发现执行条件不满足的，他会调用 wait () 方法。这时，该线程会释放锁，并进入这个对象的等待队列，暂停执行。

### 2. 等待通知示例代码
```java
public class ProducerConsumer {
    private final Object lock = new Object(); // 共享锁对象
    private int count = 0; // 共享资源
    private final int LIMIT = 5; // 资源上限

    // 生产者方法
    public void produce() throws InterruptedException {
        synchronized (lock) {
            // 必须用while循环检查条件
            while (count >= LIMIT) {
                System.out.println("缓冲区已满，生产者等待...");
                lock.wait(); // 1. 释放lock锁 2. 进入等待 3. 被唤醒后从这返回
            }
            // 条件满足，生产数据
            count++;
            System.out.println("生产后，当前数量: " + count);
            // 生产完成后，通知消费者（唤醒一个等待的消费者线程）
            lock.notify();
            // lock.notifyAll(); // 在多生产者多消费者场景下更安全
        }
    }

    // 消费者方法
    public void consume() throws InterruptedException {
        synchronized (lock) {
            // 必须用while循环检查条件
            while (count <= 0) {
                System.out.println("缓冲区为空，消费者等待...");
                lock.wait(); // 1. 释放lock锁 2. 进入等待 3. 被唤醒后从这返回
            }
            // 条件满足，消费数据
            count--;
            System.out.println("消费后，当前数量: " + count);
            // 消费完成后，通知生产者（唤醒一个等待的生产者线程）
            lock.notify();
            // lock.notifyAll(); // 在多生产者多消费者场景下更安全
        }
    }
}
```

### 3. 关键注意事项与最近实践
1. **必须在同步块中调用**：这是硬性规定。`wait()` 和 `notify()` 的调用必须位于 `synchronized` 代码块或方法内，且调用它们的对象必须与 `synchronized` 锁定的对象是**同一个**，否则会抛出 `IllegalMonitorStateException
2. **始终在循环中检查条件**：这是防御**虚假唤醒**的最佳实践。虚假唤醒是指线程在没有收到明确通知的情况下被唤醒。使用 `if` 判断可能会导致条件未满足就继续执行，从而引发逻辑错误 
3. **notify() 与 notifyAll() 的选择**：
    - `notify()`随机唤醒等待队列中的一个线程，效率较高，但风险在于可能唤醒的不是当前所需条件的线程，导致“信号丢失”或某些线程长期“饥饿”
    - `notifyAll()` 会唤醒所有等待的线程，让它们都去竞争锁并重新检查条件。这更安全，但会带来一定的性能开销。在复杂的多条件等待场景下，通常更推荐使用 `notifyAll()`
 4. **考虑使用 java. util. concurrent 工具**：对于复杂的并发场景，Java 提供的 `java.util.concurrent` 包下的高级工具（如 `ReentrantLock` 及其关联的 `Condition` 对象）能提供更灵活、更强大的线程通信能力，例如支持多个等待条件队列

