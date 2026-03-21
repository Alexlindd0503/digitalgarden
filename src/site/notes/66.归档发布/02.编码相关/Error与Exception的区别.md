---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Error与Exception的区别/"}
---

#java #最佳实践 #面试

```ad-summary
title: 总结

- `Error` 和 `Exception` 都继承自 `Throwable`，但定位完全不同
- `Error` 代表 JVM 级别的严重故障，不应捕获；`Exception` 是业务层面的可预期异常
- `Exception` 分两类：checked（编译期强制处理）和 unchecked（运行时异常）
- 三条实践原则：
	- 捕获具体异常
	- 不要生吞异常
	- Throw early catch late
- 异常实例化会做栈快照，高频场景下性能开销不可忽视
```

## 1. 继承体系

`Error` 和 `Exception` 都继承自 `Throwable`。Java 中只有 `Throwable` 的实例才能被 `throw` 抛出或 `catch` 捕获，这是整个异常处理机制的基础。

```
Throwable
├── Error          // JVM 级别严重错误，不可恢复
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── ...
└── Exception      // 程序层面的可预期异常
    ├── IOException          // checked
    ├── SQLException         // checked
    └── RuntimeException     // unchecked
        ├── NullPointerException
        ├── ArrayIndexOutOfBoundsException
        └── ...
```

## 2. Error vs Exception

| | Error | Exception |
|---|---|---|
| 定位 | JVM / 系统级别严重故障 | 程序运行中可预期的意外 |
| 可恢复性 | 通常不可恢复 | 可以捕获并处理 |
| 是否应捕获 | 不应该 | 应该，并做相应处理 |
| 典型例子 | `OutOfMemoryError`、`StackOverflowError` | `IOException`、`NullPointerException` |

`Error` 代表的是 JVM 自身处于非正常状态，比如堆内存耗尽、栈溢出，这类情况程序本身无力回天，捕获了也没有意义。

## 3. checked vs unchecked

`Exception` 进一步分为两类：

**checked 异常（可检查异常）**
- 编译期强制要求处理，不处理无法通过编译
- 代表调用方必须考虑的外部风险，如文件不存在、网络中断
- 典型：`IOException`、`SQLException`

**unchecked 异常（运行时异常）**
- 继承自 `RuntimeException`，编译器不强制处理
- 通常是代码逻辑错误，应该通过修复代码来避免，而不是捕获
- 典型：`NullPointerException`、`ArrayIndexOutOfBoundsException`、`IllegalArgumentException`

## 4. 最佳实践

### 捕获具体异常，不要一把抓

```java
// 不推荐：吞掉了所有异常，问题被掩盖
try {
    // 业务代码
    Thread.sleep(1000L);
} catch (Exception e) {
    // Ignore it  ← 这是在埋雷
}

// 推荐：捕获你真正预期的异常
try {
    Thread.sleep(1000L);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 恢复中断状态
    log.warn("线程被中断", e);
}
```

捕获 `Exception` 这类通用异常会把你没预料到的问题也一起吞掉，排查时会非常痛苦。

### 不要生吞异常

```java
// 不推荐：异常消失了，没有任何痕迹
try {
    // 业务代码
} catch (IOException e) {
    e.printStackTrace(); // 只打印到 stderr，生产环境可能根本看不到
}

// 推荐：记录日志或向上抛出
try {
    // 业务代码
} catch (IOException e) {
    log.error("读取配置文件失败", e);
    throw new RuntimeException("系统初始化失败", e); // 保留原始 cause
}
```

生吞异常会让程序在后续以不可控的方式崩溃，届时堆栈信息完全对不上，排查成本极高。

### Throw early，catch late

**Throw early**：尽早暴露问题，不要让非法状态传播。

```java
// 不推荐：fileName 为 null 时，NullPointerException 会在深处抛出
public void readPreferences(String fileName) {
    InputStream in = new FileInputStream(fileName); // 这里才炸
}

// 推荐：入口处立即校验，堆栈信息清晰明了
public void readPreferences(String filename) {
    Objects.requireNonNull(filename, "filename 不能为 null");
    InputStream in = new FileInputStream(filename);
}
```

**Catch late**：不知道怎么处理就不要捕获，让异常继续向上传播，在有足够业务上下文的地方统一处理。实在要捕获，也要保留原始 `cause`：

```java
catch (IOException e) {
    throw new ServiceException("用户数据加载失败", e); // 不要丢掉 e
}
```

## 5. 性能注意事项

异常机制有两处潜在的性能开销：

**try-catch 块影响 JIT 优化**
`try-catch` 会限制 JVM 对代码块的优化，建议只包裹真正可能抛出异常的代码，不要用一个大 `try` 包住整个方法体。

**用异常控制流程是反模式**
```java
// 不推荐：用异常做流程控制，比 if/else 慢几十倍
try {
    int value = Integer.parseInt(input);
} catch (NumberFormatException e) {
    // 当作非数字处理
}

// 推荐
if (input.matches("\\d+")) {
    int value = Integer.parseInt(input);
}
```

**实例化异常会做栈快照**
每次 `new Exception()` 都会捕获当前完整的调用栈，这是一个相对重的操作。如果某个异常在高频路径上被频繁抛出，这个开销会显著影响吞吐量。对于性能敏感的场景，可以考虑复用异常对象或使用不带栈信息的轻量异常（覆写 `fillInStackTrace()` 返回 `this`）。
