---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/Java异常处理最佳实践/"}
---

#java #最佳实践

## 1. 先说结论

几条核心原则：
- 捕获特定异常，别捕获 `Exception`
- 不生吞异常，至少记日志
- 早抛出，晚捕获
- 业务异常用 unchecked（`RuntimeException`），别用 checked

## 2. 捕获特定异常

```java
// 别这么写，太宽泛，会把不该捕获的异常也吃掉
try {
    Thread.sleep(1000L);
} catch (Exception e) { }

// 应该这么写
try {
    Thread.sleep(1000L);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // 恢复中断标志
    throw new RuntimeException("线程被中断", e);
}
```

## 3. 不生吞异常

```java
// 最差的写法，出了问题完全不知道
try {
    doSomething();
} catch (IOException e) {
    // 什么都不做
}

// 至少记日志，保留完整堆栈（注意要传 e，不是 e.getMessage()）
try {
    doSomething();
} catch (IOException e) {
    log.error("处理失败", e);
    throw new BizException("处理失败", e); // 保留异常链
}
```

## 4. 早抛出，晚捕获

**早抛出**：入口处就检查参数，快速失败，方便定位问题。

```java
// 别等到深处才报 NPE，调用栈一长就难找了
public void readFile(String filename) {
    Objects.requireNonNull(filename, "filename 不能为空");
    // 后续逻辑...
}
```

**晚捕获**：底层不知道怎么处理就别捕获，让异常往上抛，在有业务上下文的地方统一处理。

```java
// DAO 层：直接抛，不处理
public User findById(Long id) throws DataAccessException { ... }

// Service 层：转成业务异常
public User getUser(Long id) {
    try {
        return userDao.findById(id);
    } catch (DataAccessException e) {
        throw new BizException("用户查询失败", e);
    }
}

// Controller 层：交给全局异常处理器，不用每个接口都 catch
```

## 5. 业务异常用 unchecked

checked 异常（非 RuntimeException）会强迫调用链每一层都处理或声明，污染代码。Spring 框架本身也全用 unchecked，Effective Java 也建议避免不必要的 checked 异常。

```java
// 自定义业务异常，继承 RuntimeException
public class BizException extends RuntimeException {
    private final String code;

    public BizException(String code, String message) {
        super(message);
        this.code = code;
    }

    public BizException(String message, Throwable cause) {
        super(message, cause);
        this.code = "SYSTEM_ERROR";
    }
}
```

## 6. 全局异常处理

别在每个 Controller 里各自 catch，用 `@ControllerAdvice` 统一处理：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 业务异常：返回具体错误信息
    @ExceptionHandler(BizException.class)
    public Result<Void> handleBizException(BizException e) {
        log.warn("业务异常: {}", e.getMessage());
        return Result.fail(e.getCode(), e.getMessage());
    }

    // 参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidException(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.joining(", "));
        return Result.fail("PARAM_ERROR", msg);
    }

    // 兜底：未知异常不暴露内部细节
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail("SYSTEM_ERROR", "系统繁忙，请稍后重试");
    }
}
```

## 7. 几个注意点

- `try-catch` 块别包太大，只包真正可能抛异常的那几行
- 别用异常控制业务流程（比如用 `ArrayIndexOutOfBoundsException` 结束循环），性能差且难读
- 记日志一定要传异常对象 `log.error("msg", e)`，只记 `e.getMessage()` 会丢失堆栈
- 高频路径上频繁抛异常会有性能问题，因为每次创建异常都要做栈快照
