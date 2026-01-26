---
{"dg-publish":true,"permalink":"/01.后端技术学习/并发/Lock 与 Condition 机制/"}
---

### 1. 📋 概述
Lock 与 Condition 机制是 Java 并发包中提供的同步原语，**用于替代传统的 synchronized 关键字，提供更灵活、更强大的线程同步控制能力**。

---

### 2. 🔒 Lock 接口详解
#### 2.1. 基本概念
- **Lock**：用于解决 synchronized **申请不到资源（进入休眠）不释放**已占用资源的问题，能响应中断、支持超时、非阻塞获取锁
- 利用内部 volatile 变量 state 保障可见性
- 是可重入锁，有公平锁与非公平锁之分
- 公平锁会唤醒等待时间最长的线程

#### 2.2. 核心功能特性
##### 2.2.1. **响应中断获取锁**
```java
lock.lockInterruptibly();
```
- 当获取不到锁时能响应中断信号
- 避免线程无限期等待

##### 2.2.2. **超时获取锁**
```java
tryLock(Long time, TimeUnit unit)
```
- 在指定时间内尝试获取锁
- 超时后自动返回，不会一直阻塞

##### 2.2.3. **非阻塞获取锁**
```java
tryLock()
```
- 尝试非阻塞地获取锁
- 立即返回获取结果（true/false）

---

### 3. 🔄 Condition 机制详解

#### 3.1. 基本概念
- **Condition**：是管程中的条件变量的队列
- Lock + Condition 可实现异步转同步 - RPC 调用
- 提供了比 Object 监视器更灵活的线程等待/唤醒机制

#### 3.2. 核心方法
- `await()`：使当前线程等待
- `signal()`：唤醒单个等待线程
- `signalAll()`：唤醒所有等待线程

---

### 4. 💡 完整代码示例
```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class LockConditionExample {
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();
    private boolean flag = false;

    /**
     * 生产者方法
     */
    public void produce() throws InterruptedException {
        lock.lock();
        try {
            while (flag) {
                condition.await();  // 等待条件满足
            }
            // 生产逻辑
            System.out.println("生产产品...");
            flag = true;
            condition.signal();   // 唤醒消费者
        } finally {
            lock.unlock();  // 确保锁释放
        }
    }

    /**
     * 消费者方法
     */
    public void consume() throws InterruptedException {
        lock.lock();
        try {
            while (!flag) {
                condition.await();  // 等待条件满足
            }
            // 消费逻辑
            System.out.println("消费产品...");
            flag = false;
            condition.signal();   // 唤醒生产者
        } finally {
            lock.unlock();  // 确保锁释放
        }
    }

    /**
     * 演示中断响应
     */
    public void interruptibleLockExample() throws InterruptedException {
        lock.lockInterruptibly();  // 可中断的锁获取
        try {
            // 业务逻辑
        } finally {
            lock.unlock();
        }
    }

    /**
     * 演示超时锁获取
     */
    public boolean tryLockWithTimeout() throws InterruptedException {
        if (lock.tryLock(5, TimeUnit.SECONDS)) {
            try {
                // 业务逻辑
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
}
```

---

### 5. ⚠️ 重要注意事项
#### 5.1. Lock 使用注意事项
1. **正确的加锁和解锁操作**：确保资源的正确释放
2. **避免死锁**：合理设计锁的获取顺序
3. **异常处理**：在 finally 块中释放锁

#### 5.2. Condition 使用注意事项
1. **正确的 Lock 配合**：await () 等方法调用前必须持有相应的 Lock
2. **合适的信号发送时机**：在合适的时候发送 signal/signalAll 信号
3. **避免虚假唤醒**：使用 while 循环检查条件而非 if

---

### 6. 🆚 Lock vs Synchronized 对比

| 特性    | Lock                      | synchronized |
| ----- | ------------------------- | ------------ |
| 中断响应  | ✅ 支持 lockInterruptibly () | ❌ 不支持        |
| 超时机制  | ✅ 支持 tryLock (time)       | ❌ 不支持        |
| 非阻塞获取 | ✅ tryLock ()              | ❌ 不支持        |
| 公平性控制 | ✅ 可配置公平/非公平               | ❌ 非公平        |
| 条件变量  | ✅ 支持多个 Condition          | ❌ 只有一个等待队列   |
| 使用复杂度 | ⚠️ 相对复杂                   | ✅ 简单自动       |

---

### 7. 🎯 最佳实践总结
1. **锁获取模式选择**：
   - 简单场景使用 lock ()
   - 需要中断响应使用 lockInterruptibly ()
   - 需要超时控制使用 tryLock (time)

2. **资源释放保证**：
   - 始终在 finally 块中 unlock ()
   - 避免嵌套锁导致死锁

3. **条件变量使用**：
   - 使用 while 循环检查条件
   - 合理选择 signal () vs signalAll ()
   - 避免过早或过晚发送信号

4. **性能考虑**：
   - 高竞争场景优先选择 Lock
   - 简单同步需求可考虑 synchronized

---

通过合理使用 Lock 与 Condition 机制，可以构建更高效、更灵活的并发程序，特别是在需要精细化控制线程同步的复杂场景下。
