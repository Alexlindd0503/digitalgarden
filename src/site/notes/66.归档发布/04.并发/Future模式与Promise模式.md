---
{"dg-publish":true,"permalink":"/66.归档发布/04.并发/Future模式与Promise模式/"}
---

#并发

```ad-summary
title: 总结

- Future 模式：提交任务拿占位符，稍后 get() 或回调获取结果
- CompletableFuture（JDK 8）是 Promise 模式，链式调用解决回调地狱，生产首选
- 生产环境给 CompletableFuture 传自定义线程池，别用默认的 ForkJoinPool
```

## 核心概念

**Future 模式**：代表一个操作未来的结果（占位符），分两种子模式：
- **将来式**：提交任务后拿到 future，稍后调用 `get()` 获取结果
- **回调式**：任务完成后自动触发回调，主线程不阻塞

**Promise 模式**：解决回调地狱（Callback Hell）问题，将嵌套回调改写为链式调用

## Skill 1：JDK Future（将来式）

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(() -> {
    // 耗时操作
    return 100;
}); // 非阻塞，立即返回

// 可在此处做其他事情（并行效果）

Integer result = future.get(); // 阻塞直到结果就绪
```

**JDK Future 接口方法：**

| 方法 | 说明 |
|------|------|
| `get()` | 阻塞等待结果 |
| `get(timeout, unit)` | 带超时的阻塞等待 |
| `isDone()` | 非阻塞轮询是否完成 |
| `cancel(mayInterrupt)` | 取消任务 |
| `isCancelled()` | 是否已取消 |

**适用场景**：多个耗时操作并发执行，提交后各自 `get()`。

## Skill 2：回调式 Future（Netty）

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.22.Final</version>
</dependency>
```

```java
EventExecutorGroup group = new DefaultEventExecutorGroup(4);
Future<Integer> f = group.submit(() -> {
    // 耗时操作
    return 100;
});
f.addListener(future -> {
    System.out.println("结果：" + future.get()); // 任务完成后自动触发
});
// 主线程不阻塞
```

## Skill 3：回调式 Future（Guava ListenableFuture）

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>21.0</version>
</dependency>
```

```java
ListeningExecutorService service = MoreExecutors.listeningDecorator(Executors.newSingleThreadExecutor());
ListenableFuture<Integer> future = service.submit(() -> {
    // 耗时操作
    return 100;
});
Futures.addCallback(future, new FutureCallback<Integer>() {
    public void onSuccess(Integer result) { /* 成功处理 */ }
    public void onFailure(Throwable t)    { /* 失败处理 */ }
});
// 主线程不阻塞
```

> 无论何时 addListener/addCallback，都能正确接收到回调，不存在错过的问题。

## Skill 4：CompletableFuture（JDK 8，Promise 模式）

无需第三方依赖，解决回调地狱，支持链式调用。

**基本回调：**

```java
CompletableFuture.supplyAsync(() -> {
    // 耗时操作
    return 100;
}).whenComplete((result, e) -> {
    System.out.println("结果：" + result);
});
// 主线程不阻塞
```

**链式回调（解决回调地狱）：**

```java
CompletableFuture.supplyAsync(() -> {
    // 第一步耗时操作
    return 100;
}).thenCompose(i ->
    CompletableFuture.supplyAsync(() -> {
        // 第二步耗时操作，依赖第一步结果
        return i + 100;
    })
).whenComplete((result, e) -> {
    System.out.println("最终结果：" + result); // 200
});
```

**常用方法：**

| 方法 | 说明 |
|------|------|
| `supplyAsync(supplier)` | 异步执行有返回值的任务 |
| `runAsync(runnable)` | 异步执行无返回值的任务 |
| `whenComplete(biConsumer)` | 完成后回调（含异常） |
| `thenCompose(fn)` | 链式异步任务（扁平化） |
| `thenApply(fn)` | 对结果做同步转换 |
| `thenAccept(consumer)` | 消费结果，无返回值 |

## 关系总结

```
Future 模式
├── 将来式：submit → get()（可能阻塞）
└── 回调式：addListener / addCallback（不阻塞）
        └── 多层嵌套 → Callback Hell
                └── Promise 模式（CompletableFuture 链式调用）解决
```

## 相关链接

- [[02.并发工具类#Future 与 CompletableFuture\|02.并发工具类#Future 与 CompletableFuture]] — 工具类使用入口
- [[66.归档发布/04.并发/线程池ThreadPoolExecutor\|线程池ThreadPoolExecutor]] — CompletableFuture 默认使用 ForkJoinPool，生产建议自定义线程池
