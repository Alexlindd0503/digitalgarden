---
{"dg-publish":true,"permalink":"/66.归档发布/04.并发/Volatile关键字的底层原理/","dg-note-properties":{"时间":"2026-03-17"}}
---

#java #并发 #原理

```ad-summary
title: 总结

- volatile 保证可见性（lock 指令 + MESI 协议）和有序性（内存屏障），但不保证原子性
- 双重检查单例必须加 volatile，防止 `new` 操作指令重排导致拿到未初始化的对象
- 状态标志位、发布不可变对象用 volatile；需要原子操作用 AtomicInteger；需要互斥用锁
```

## 1. volatile 解决什么问题？

volatile 用来解决**可见性**和**有序性**，但**不保证原子性**。

对应 [[Java的内存模型\|Java的内存模型]] 里的两个问题：
- 可见性：一个线程改了变量，另一个线程不一定能立刻看到
- 有序性：编译器可能对指令重排，导致执行顺序和代码顺序不一致

## 2. 可见性怎么保证的？

线程 A 写了一个 volatile 变量，线程 B 去读，能立刻拿到最新值。

底层靠两件事：

**① lock 前缀指令**：线程 A 写 volatile 变量时，JVM 会给 CPU 发一条 `lock` 前缀指令，强制把当前 CPU 缓存行的数据写回主内存。

**② MESI 缓存一致性协议**：其他 CPU 一直在嗅探总线，发现主内存里这个地址的数据被改了，就把自己缓存里对应的数据标记为**失效（Invalid）**。线程 B 下次读这个变量，发现缓存失效，就重新从主内存加载。

```java
// 没有 volatile：线程 B 可能永远读到 running = true，死循环
// 加了 volatile：线程 A 改完，线程 B 能立刻感知到
private volatile boolean running = true;

// 线程 A
running = false;

// 线程 B
while (running) {
    // 做事情
}
```

## 3. 有序性怎么保证的？

靠**内存屏障**（Memory Barrier）禁止指令重排。

JVM 在 volatile 读写前后插入屏障：

| 操作 | 插入屏障 | 效果 |
|------|---------|------|
| volatile 写之前 | StoreStore | 前面的普通写不能排到 volatile 写后面 |
| volatile 写之后 | StoreLoad | volatile 写不能排到后面的读前面 |
| volatile 读之后 | LoadLoad + LoadStore | 后面的读写不能排到 volatile 读前面 |

最典型的例子是**双重检查单例**，没有 volatile 就会出问题：

```java
public class Singleton {
    // 必须加 volatile，否则可能拿到未初始化的对象
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {              // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {      // 第二次检查
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

`new Singleton()` 在底层分三步：
1. 分配内存
2. 初始化对象
3. 把引用指向内存地址

没有 volatile，编译器可能把步骤 2 和 3 对调（1→3→2）。这样线程 A 刚执行完步骤 3，线程 B 进来一看 `instance != null`，直接返回了，但对象还没初始化完，用起来就崩了。

加了 volatile，禁止重排，就不会有这个问题。

## 4. volatile 不能保证原子性

volatile 只管可见性和有序性，**复合操作还是不安全的**。

```java
private volatile int count = 0;

// ❌ 这样不行，i++ 是三步操作：读 → 加 → 写
// 多线程下还是会丢数据
count++;

// ✅ 需要原子操作，用 AtomicInteger
private AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```

`count++` 看起来一行，实际上是三条指令：读取 count → 加 1 → 写回。线程切换可能发生在任意两步之间，volatile 管不了这个。

## 5. 什么时候用 volatile？

适合用的场景：
- **状态标志位**：一个线程写，多个线程读（如上面的 `running` 例子）
- **双重检查单例**：配合 synchronized 使用
- **发布不可变对象**：确保对象引用对其他线程可见

不适合用的场景：
- 需要原子操作（用 `AtomicInteger` 等原子类，底层是 [[66.归档发布/04.并发/CAS学习\|CAS]]）
- 需要互斥（用 `synchronized` 或 `ReentrantLock`，见 [[66.归档发布/04.并发/AQS抽象队列同步器原理\|AQS抽象队列同步器原理]]）
