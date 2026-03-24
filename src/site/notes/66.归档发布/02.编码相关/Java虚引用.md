---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Java虚引用/","dg-note-properties":{"时间":"2026-03-15"}}
---

#java #jvm #引用 #gc

## 1. 虚引用是什么？

Java 有四种引用强度：强引用 > 软引用 > 弱引用 > 虚引用。虚引用是最弱的一种，弱到对对象的生命周期完全没有影响。

```java
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> ref = new PhantomReference<>(obj, queue);

ref.get(); // 永远返回 null
```

`get()` 永远返回 `null`，你没法通过虚引用拿到对象。虚引用必须配合 `ReferenceQueue` 使用，否则没有任何意义。

## 2. 设计初衷

虚引用的唯一用途是：**监听对象被 GC 回收这个事件，并在回收后执行清理操作**。

当 GC 准备回收一个对象时，如果发现它还关联着虚引用，就会在回收后把这个虚引用放入 `ReferenceQueue`。程序通过轮询队列感知到这个事件，然后做相应的资源清理。

这解决了 `finalize()` 的问题。`finalize()` 也能在对象回收前做清理，但它有严重缺陷：执行时机不确定、可能让对象"复活"、性能差。虚引用 + `ReferenceQueue` 是更可靠的替代方案。

JDK 中 `DirectByteBuffer` 的堆外内存释放就是用这个机制实现的，详见 [[66.归档发布/02.编码相关/JAVA的堆外内存\|JAVA的堆外内存]]。

## 3. 怎么用？

基本模式：一个线程持续轮询 `ReferenceQueue`，有对象被回收就触发清理逻辑。

```java
public class ResourceHolder {
    // 引用队列，GC 回收对象后虚引用会进入这里
    private static final ReferenceQueue<Object> QUEUE = new ReferenceQueue<>();

    // 继承 PhantomReference，附带清理逻辑
    static class CleanableRef extends PhantomReference<Object> {
        private final String resourceName;

        CleanableRef(Object referent, String resourceName) {
            super(referent, QUEUE);
            this.resourceName = resourceName;
        }

        void cleanup() {
            System.out.println("释放资源: " + resourceName);
            // 实际场景：关闭文件句柄、释放堆外内存等
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Object obj = new Object();
        CleanableRef ref = new CleanableRef(obj, "my-resource");

        // 让 obj 变成可回收状态
        obj = null;

        // 轮询队列（实际项目中放到独立线程里）
        System.gc();
        Reference<?> polled = QUEUE.remove(); // 阻塞等待
        if (polled instanceof CleanableRef) {
            ((CleanableRef) polled).cleanup();
        }
    }
}
```

实际项目里轮询逻辑放到独立的守护线程里，持续处理：

```java
Thread cleanerThread = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            Reference<?> ref = QUEUE.remove(); // 阻塞等待
            if (ref instanceof CleanableRef) {
                ((CleanableRef) ref).cleanup();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
});
cleanerThread.setDaemon(true);
cleanerThread.start();
```

## 4. 注意事项

**必须配合 ReferenceQueue**，否则 GC 回收后没有任何通知，虚引用毫无意义。

**清理后要显式 clear**，不移除的话对象不会被彻底销毁，相当于内存泄漏：

```java
Reference<?> ref = QUEUE.poll();
if (ref != null) {
    doCleanup();
    ref.clear(); // 显式清除，让对象可以被彻底回收
}
```

**不要在清理逻辑里访问原对象**，`get()` 永远返回 `null`，清理所需的资源信息要在构造虚引用时就保存好（比如文件路径、内存地址等）。

**GC 时机不确定**，虚引用的回调依赖 GC，不能用于有严格时序要求的资源释放。需要确定性释放的场景，用 `try-with-resources` 更合适。

**JDK 9+ 推荐用 `Cleaner`**，它是对虚引用 + ReferenceQueue 模式的封装，更安全易用：

```java
Cleaner cleaner = Cleaner.create();
Object obj = new Object();
cleaner.register(obj, () -> System.out.println("对象被回收，执行清理"));
```

## 5. 经典应用场景

### 5.1 DirectByteBuffer 堆外内存释放

这是 JDK 里最典型的虚引用应用。`ByteBuffer.allocateDirect()` 分配堆外内存时，会同时创建一个 `Cleaner` 对象（虚引用子类）绑定到 `DirectByteBuffer`。

```
DirectByteBuffer（堆内）──虚引用──▶ Cleaner
       │
       └── 持有堆外内存地址
```

当 `DirectByteBuffer` 被 GC 回收，`Cleaner` 进入 `ReferenceQueue`，JDK 内部的 `ReferenceHandler` 线程检测到后调用 `Cleaner.clean()`，最终执行 `unsafe.freeMemory()` 释放堆外内存。整个过程对使用者完全透明。

### 5.2 JDK 9+ 的 java.lang.ref.Cleaner

JDK 9 把这套模式封装成了公开 API `java.lang.ref.Cleaner`，替代了不安全的 `finalize()`：

```java
public class NativeResource implements AutoCloseable {
    private static final Cleaner CLEANER = Cleaner.create();

    private final Cleaner.Cleanable cleanable;
    private final long nativeHandle; // 假设持有一个 native 资源句柄

    public NativeResource() {
        this.nativeHandle = allocateNative(); // 分配 native 资源
        // 注册清理动作，注意 lambda 里不能捕获 this，否则强引用导致永远不被回收
        long handle = this.nativeHandle;
        this.cleanable = CLEANER.register(this, () -> releaseNative(handle));
    }

    @Override
    public void close() {
        cleanable.clean(); // 主动释放，不等 GC
    }

    private static long allocateNative() { return 0L; /* 省略 */ }
    private static void releaseNative(long handle) { /* 释放 native 资源 */ }
}
```

实现了 `AutoCloseable`，正常情况下 `try-with-resources` 主动释放；忘记关闭时，GC 兜底触发清理。这是目前推荐的资源管理模式。

### 5.3 Netty 的 ResourceLeakDetector

Netty 用虚引用实现了内存泄漏检测。每次分配 `ByteBuf` 时，会创建一个 `DefaultResourceLeak`（虚引用）跟踪这个对象：

```
ByteBuf 分配 ──▶ 创建 DefaultResourceLeak（虚引用）
                        │
                        └── 记录分配时的调用栈
```

如果 `ByteBuf` 被 GC 回收时还没有调用 `release()`，`DefaultResourceLeak` 进入队列，Netty 检测到后打印警告日志，并附上分配时的调用栈，帮助定位泄漏位置。

这就是为什么 Netty 应用里有时会看到这样的日志：

```
LEAK: ByteBuf.release() was not called before it's garbage-collected.
```

### 5.4 WeakHashMap 的近亲用法

`WeakHashMap` 用的是弱引用而不是虚引用，但原理类似：key 被 GC 回收后，对应的 Entry 会进入 `ReferenceQueue`，下次操作 map 时自动清理过期 Entry。

虚引用和弱引用的区别在于：弱引用的 `get()` 在对象被回收前还能拿到对象，虚引用的 `get()` 永远返回 `null`。需要在清理时访问原对象就用弱引用，只需要感知"对象已死"这个事件就用虚引用。

## 相关链接

- [[66.归档发布/02.编码相关/JAVA的堆外内存\|JAVA的堆外内存]]
- [[66.归档发布/02.编码相关/JVM性能调优方法\|JVM性能调优方法]]
