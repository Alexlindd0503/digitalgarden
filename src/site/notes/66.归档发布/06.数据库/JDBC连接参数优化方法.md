---
{"dg-publish":true,"permalink":"/66.归档发布/06.数据库/JDBC连接参数优化方法/","dg-note-properties":{"时间":"2026-03-14"}}
---

#数据库 #最佳实践 #mysql #java

```ad-summary
title: 结论
JDBC 连接参数里最值得关注的就一个：`rewriteBatchedStatements=true`，批量插入性能能提升几十倍。其他参数基本是字符集、时区、超时的标准配置。
```
## 1 关键参数

### 1.1 rewriteBatchedStatements（批量操作必开）

把多条 INSERT 合并成一条，减少网络往返：

```sql
-- 未开启：3 次网络通信
INSERT INTO user (name, age) VALUES ('zhang', 20);
INSERT INTO user (name, age) VALUES ('li', 25);
INSERT INTO user (name, age) VALUES ('wang', 30);

-- 开启后：1 次网络通信
INSERT INTO user (name, age) VALUES ('zhang', 20), ('li', 25), ('wang', 30);
```

批量插入 1 万条数据，开启前后耗时差距在 10-50 倍左右（实际效果取决于网络和机器）。

配合代码这么用：

```java
String sql = "INSERT INTO user (name, age) VALUES (?, ?)";
PreparedStatement ps = conn.prepareStatement(sql);

for (int i = 0; i < 10000; i++) {
    ps.setString(1, "user" + i);
    ps.setInt(2, 20 + i);
    ps.addBatch();

    if (i % 500 == 0) {
        ps.executeBatch();
        ps.clearBatch();
    }
}
ps.executeBatch();
```

### 1.2 字符集

新项目统一用 `utf8mb4`，支持 emoji，`utf8` 只有 3 字节不支持：

```
useUnicode=true&characterEncoding=utf8mb4
```

数据库、表、字段、连接字符集要保持一致，不然还是会乱码。

### 1.3 时区

```
serverTimezone=Asia/Shanghai
```

不设的话 MySQL 8 会报 `The server time zone value 'CST' is unrecognized`。跨时区系统建议统一用 UTC 存储，展示时再转换。

### 1.4 SSL

```
useSSL=false   # 内网，不需要加密，减少 CPU 开销
useSSL=true    # 公网，必须开
```

### 1.5 预编译缓存

```
cachePrepStmts=true&prepStmtCacheSize=250&useServerPrepStmts=true
```

缓存预编译语句，高频查询场景有效果。

### 1.6 超时设置

```
connectTimeout=10000&socketTimeout=30000
```

一定要设，不然连接挂起会一直等。

## 2 完整配置

开发环境：

```
jdbc:mysql://localhost:3306/mydb?rewriteBatchedStatements=true&serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=utf8mb4&useSSL=false&logSlowQueries=true&slowQueryThresholdMillis=1000
```

生产环境：

```
jdbc:mysql://db.example.com:3306/mydb?rewriteBatchedStatements=true&serverTimezone=UTC&useUnicode=true&characterEncoding=utf8mb4&useSSL=true&cachePrepStmts=true&prepStmtCacheSize=250&useServerPrepStmts=true&connectTimeout=10000&socketTimeout=30000
```

## 3 几个注意点

- `allowMultiQueries=true` 允许一次执行多条 SQL，有 SQL 注入风险，生产环境别开，用事务代替
- `autoReconnect=true` 不推荐，会掩盖连接问题，连接管理交给 HikariCP 这类连接池来做
- MySQL 5.x 和 8.x 部分参数有差异，升级驱动版本时注意核对官方文档

