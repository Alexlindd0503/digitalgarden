---
{"dg-publish":true,"permalink":"/66.归档发布/02.编码相关/JVM性能调优方法/","dg-note-properties":{"时间":"2026-03-14"}}
---

#java #最佳实践 #jvm

```ad-summary
title: 总结

- 调优核心目标：减少 Full GC 频率，降低停顿时间，让对象尽量在新生代回收
- 调优优先级：先改代码 → 合理分配内存 → 选对 GC 器 → 最后才微调参数
- 现在基本 G1 起步，大内存上 ZGC；G1 不需要手动设 `-Xmn`
- OOM 必备：`-XX:+HeapDumpOnOutOfMemoryError`，出了问题才有得查
```

## 1. 先说结论

JVM 调优的核心目标就一个：**减少 Full GC 频率，降低 GC 停顿时间**。

让对象尽量在新生代回收，别让它们跑到老年代去。

调优优先级：
1. 先看代码，减少对象创建、修内存泄漏
2. 合理分配堆内存和分代比例
3. 选对 GC 器（现在基本上 G1 起步，大内存上 ZGC）
4. 最后才是参数微调

## 2. 监控工具

### 2.1 jstat 看 GC 情况

```bash
# 每 1000ms 打印一次，共 10 次
jstat -gc <PID> 1000 10
```

关键字段：
- `YGC` / `YGCT`：Young GC 次数和总耗时
- `FGC` / `FGCT`：Full GC 次数和总耗时
- `EU`：Eden 使用量，看增长速率
- `OU`：老年代使用量，持续增长说明有泄漏

```bash
jstat -gcnew <PID>      # 看新生代，含 TT（晋升年龄）
jstat -gcold <PID>      # 看老年代
jstat -class <PID>      # 看类加载，排查 Metaspace 问题
```

### 2.2 jmap 看内存分布

```bash
# 看对象分布，按内存大小排序
jmap -histo <PID>

# 生成堆快照，用 MAT 离线分析
jmap -dump:live,format=b,file=dump.hprof <PID>
```

MAT（Eclipse Memory Analyzer）比 `jhat` 强很多，排查内存泄漏首选。

### 2.3 开启 GC 日志

```bash
# JDK 8
-XX:+PrintGCDetails -Xloggc:gc.log -XX:+PrintGCDateStamps

# JDK 9+
-Xlog:gc*:file=gc.log:time,uptime
```

日志里重点看：
- `Allocation Failure`：Eden 满了触发 Young GC，正常
- `Concurrent Mode Failure`：CMS 并发回收没跑完老年代就满了，说明 CMS 触发太晚
- `Promotion Failed`：晋升失败，Survivor 或老年代放不下了

## 3. 内存参数怎么设？

### 3.1 先估算对象增长速率

```
示例：支付系统，每台机器 QPS 30
每个请求产生对象约 500 字节
每秒内存占用：30 * 500B ≈ 15KB
考虑其他对象（放大 20 倍）：约 300KB/s
```

### 3.2 根据估算设参数

4 核 8G 机器，G1 配置示例：

```bash
-Xms4G -Xmx4G
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200        # 目标停顿时间，G1 会自动调整
-XX:MetaspaceSize=256M
-XX:MaxMetaspaceSize=256M
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/dump.hprof
-Xlog:gc*:file=/logs/gc.log:time,uptime
```

G1 不需要手动设 `-Xmn`，它会自动调整新生代大小。

### 3.3 压测验证

压测时用 `jstat -gc <PID> 1000` 盯着看：
- Young GC 频率是否合理（几分钟一次比较正常）
- Young GC 后存活对象能否放进 Survivor（放不下就会提前晋升）
- Full GC 是否出现，出现了就要查原因

## 4. 频繁 Full GC 的常见原因

### 4.1 Survivor 区太小，对象提前晋升

```bash
# 用 jstat -gcnew 看 TT（实际晋升年龄）
# 如果 TT 很小（比如 1-2），说明对象还没熬到设定年龄就被迫晋升了

# 调大新生代或调整 SurvivorRatio
-XX:SurvivorRatio=4   # Eden:S0:S1 = 4:1:1，Survivor 占比更大
```

### 4.2 大对象直接进老年代

```java
// 定时任务一次性捞几百 MB 数据，直接进老年代
List<Data> all = dao.loadAll();

// 改成分批处理
int page = 0;
List<Data> batch;
while (!(batch = dao.loadPage(page++, 1000)).isEmpty()) {
    process(batch);
}
```

### 4.3 内存泄漏

老年代持续增长，Full GC 后也降不下来，基本就是泄漏。

常见场景：
```java
// 静态集合一直往里加，不清理
private static final List<Object> CACHE = new ArrayList<>();

// ThreadLocal 用完不 remove
threadLocal.set(obj);
// ... 忘了 threadLocal.remove()

// 监听器注册了不移除
eventBus.register(listener);
```

排查：`jmap -dump` 生成快照，MAT 打开，看 "Leak Suspects"。

### 4.4 代码里调了 System.gc()

```bash
# 直接禁掉
-XX:+DisableExplicitGC
```

### 4.5 Metaspace 不够

CGLib、动态代理用多了会不断生成新类，撑爆 Metaspace：

```bash
-XX:MetaspaceSize=512M
-XX:MaxMetaspaceSize=512M
```

## 5. OOM 怎么处理

| 错误信息 | 原因 | 处理 |
|---------|------|------|
| `Java heap space` | 堆内存不够或泄漏 | 加 `-XX:+HeapDumpOnOutOfMemoryError`，MAT 分析 |
| `Metaspace` | 类太多，CGLib 失控 | 增大 MetaspaceSize，检查动态代理使用 |
| `Direct buffer memory` | NIO 堆外内存泄漏 | 检查 ByteBuffer 是否及时释放 |
| `StackOverflowError` | 递归太深 | 检查递归终止条件 |

OOM 必备启动参数：

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/dump.hprof
```

出了 OOM 自动 dump，不然重启之后什么都查不到。

## 6. 什么时候需要调优？

需要动手的信号：
- Full GC 每小时超过几次
- GC 停顿超过 1 秒，接口超时
- 老年代使用率持续上涨不回落
- CPU 长期跑满，`jstat` 看到 FGC 在疯狂增长

不用动的情况：
- Young GC 几分钟一次，耗时几十 ms
- Full GC 每天就那么几次
- 响应时间正常，业务没投诉


## 相关链接

- [[01.专项学习/JavaVirtualMachine/1.JVM基础\|JVM基础]]
- [[01.专项学习/JavaVirtualMachine/3.垃圾回收\|垃圾回收]]
- [[01.专项学习/JavaVirtualMachine/5.ZGC学习\|ZGC学习]]
- [[66.归档发布/02.编码相关/JAVA的堆外内存\|JAVA的堆外内存]]
- [[66.归档发布/02.编码相关/Java虚引用\|Java虚引用]]
