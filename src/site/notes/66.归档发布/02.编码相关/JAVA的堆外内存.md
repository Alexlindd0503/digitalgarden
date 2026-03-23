---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/JAVA的堆外内存/","dg-note-properties":{"时间":"2026-03-15"}}
---

#java #jvm #内存 #nio

## 1. 堆外内存是什么？

Java 堆内存由 JVM 管理，GC 负责回收。堆外内存（Direct Memory）是在 JVM 堆之外直接向操作系统申请的内存，不受 JVM GC 管理，需要手动或通过特殊机制释放。

## 2. 为什么要用堆外内存？

**减少数据拷贝**：Java 做网络 IO 或文件读写时，堆内数据必须先拷贝到堆外内存，再交给操作系统处理。如果直接用堆外内存，就省掉了这次拷贝，这也是 Netty、RocketMQ 等框架大量使用堆外内存的原因，详见 [[66.归档发布/00.Linux/Linux中的零拷贝技术\|Linux中的零拷贝技术]]。

**降低 GC 压力**：堆外内存不在 GC 管辖范围内，大块缓存数据放堆外，可以避免频繁触发 GC，减少 Stop-The-World 停顿。

**进程间共享**：堆外内存可以在多个 JVM 实例或进程之间共享数据，堆内内存做不到这点。

## 3. 怎么用？

### 3.1 ByteBuffer.allocateDirect（推荐）

```java
// 分配 10MB 堆外内存
ByteBuffer buffer = ByteBuffer.allocateDirect(10 * 1024 * 1024);

// 写数据
buffer.put("hello".getBytes());

// 切换读模式
buffer.flip();

// 读数据
byte[] bytes = new byte[buffer.remaining()];
buffer.get(bytes);
```

堆内只存了一个很小的 `DirectByteBuffer` 对象（包含堆外内存的地址和大小），真正的数据在堆外。**不需要手动释放**，JVM 会通过 Cleaner 机制自动回收（见第 4 节）。

### 3.2 Unsafe.allocateMemory（不推荐直接用）

```java
// 通过反射拿到 Unsafe 实例
Field field = Unsafe.class.getDeclaredField("theUnsafe");
field.setAccessible(true);
Unsafe unsafe = (Unsafe) field.get(null);

// 分配 10MB 堆外内存
long address = unsafe.allocateMemory(10 * 1024 * 1024);

// 必须手动释放，否则内存泄漏
unsafe.freeMemory(address);
```

`Unsafe` 绕过了 JVM 的所有安全检查，忘记 `freeMemory` 就直接内存泄漏。Netty 内部用它是因为需要极致性能，业务代码不要直接用。

## 4. 怎么回收的？

`ByteBuffer.allocateDirect` 分配时，会同时创建一个 `Cleaner` 对象（虚引用）绑定到 `DirectByteBuffer`。

当 `DirectByteBuffer` 对象被 GC 回收时，`Cleaner` 感知到后会自动调用 `unsafe.freeMemory()` 释放对应的堆外内存。`Cleaner` 底层就是虚引用 + `ReferenceQueue` 机制，详见 [[66.归档发布/02.编码相关/Java虚引用\|Java虚引用]]。

![堆外内存示意](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/堆外内存示意.png)

问题在于：`DirectByteBuffer` 对象如果活得比较久，会晋升到老年代，只有 Old GC 或 Full GC 才能触发回收。如果长时间没有 Full GC，堆外内存就一直占着不释放，最终把物理内存耗尽。

## 5. 注意事项

**必须设置上限**，否则堆外内存无限增长：

```bash
-XX:MaxDirectMemorySize=512m
```

超过这个上限时，JVM 会触发 Full GC 尝试回收，如果还不够就抛 OOM。

**不要依赖 System.gc() 触发回收**。`ByteBuffer.allocateDirect` 内部在内存不足时会调用 `System.gc()`，但生产环境一般都加了 `-XX:+DisableExplicitGC`，这个调用直接被忽略，等于没有。

**及时释放不再使用的 DirectByteBuffer 引用**，让 GC 能尽早回收，不要让它在老年代里长期存活。

**监控堆外内存使用**：

```bash
# JVM 启动参数开启 NMT
-XX:NativeMemoryTracking=summary

# 查看堆外内存使用情况
jcmd <pid> VM.native_memory summary
```

## 相关链接

- [[66.归档发布/00.Linux/Linux中的零拷贝技术\|Linux中的零拷贝技术]]
- [[66.归档发布/02.编码相关/JVM性能调优方法\|JVM性能调优方法]]
