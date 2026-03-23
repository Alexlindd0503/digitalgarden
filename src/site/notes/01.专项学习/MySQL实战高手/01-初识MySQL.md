---
{"dg-publish":true,"permalink":"/01.专项学习/MySQL实战高手/01-初识MySQL/","dg-note-properties":{"时间":"2026-03-22"}}
---

#mysql #数据库

```ad-summary
title: 总结

- Java 应用通过 JDBC 驱动连接 MySQL，生产环境用连接池复用连接
- 一条 SQL 经过：连接器 → 分析器 → 优化器 → 执行器，最终由存储引擎执行
- InnoDB 架构分三层：内存（Buffer Pool + Change Buffer）→ 日志（redo log）→ 磁盘
- 更新操作的完整链路：undo log（回滚）→ Buffer Pool（改内存）→ redo log（崩溃恢复）→ binlog（归档）
```

## 1. 驱动与连接池

Java 应用连 MySQL 靠的是 JDBC 驱动，本质上就是 TCP 长连接。每次查询都新建连接太慢了，所以生产环境都用连接池（HikariCP、Druid 等）复用连接。

![76f983b81f133a8c7c66514cdc9642d6 MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/76f983b81f133a8c7c66514cdc9642d6_MD5.png)

连接池的配置可以看 [[66.归档发布/06.数据库/JDBC连接参数优化方法\|JDBC连接参数优化方法]]。

## 2. 一条 SQL 怎么执行的？

你写一条 `SELECT * FROM user WHERE id = 1`，MySQL 内部的执行流程大概是这样：

![f52cd1efeacf52675d54b229689f627e MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/f52cd1efeacf52675d54b229689f627e_MD5.png)

简单说就是：
1. **连接器**：先验证账号密码，看看你有没有权限
2. **分析器**：SQL 语法检查，生成解析树
3. **优化器**：决定用哪个索引、多表连接的顺序等
4. **执行器**：调用存储引擎的接口去拿数据

![f52cd1efeacf52675d54b229689f627e MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/f52cd1efeacf52675d54b229689f627e_MD5.png)

## 3. InnoDB 引擎长什么样？

MySQL 支持多种存储引擎，但日常开发基本都用 InnoDB。它的架构大概是这样：

![cf35f212f92bd3bb2ae6ab9c943338f0 MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/cf35f212f92bd3bb2ae6ab9c943338f0_MD5.png)

核心是三层结构：

**内存层**：[[66.归档发布/06.数据库/Buffer Pool配置调优\|Buffer Pool]] 是大头，数据页从磁盘加载到这里，后续读写都在内存操作。还有 Change Buffer 缓存非唯一索引的修改。

**日志层**：redo log 记录"对数据页做了什么修改"，用于崩溃恢复。WAL（Write-Ahead Logging）机制，先写日志再写磁盘，保证数据不丢。

**磁盘层**：最终的数据文件（.ibd），redo log 最终也会刷到这里。

## 4. 更新操作的完整链路

以 `UPDATE user SET name = '张三' WHERE id = 1` 为例，完整流程：

1. 加载数据页到 Buffer Pool（如果不在内存中）
2. 写 undo log，记录修改前的值，用于事务回滚
3. 更新 Buffer Pool 中的数据页
4. 写 redo log（prepare 状态）
5. 写 binlog
6. 提交事务，redo log 标记 commit

### redo log 刷盘策略

通过 `innodb_flush_log_at_trx_commit` 控制：

| 值 | 行为 | 适用场景 |
|---|------|---------|
| 0 | 每秒写入 OS cache，由 OS 决定刷盘 | 性能最好，但可能丢 1 秒数据 |
| 1 | 每次事务提交都刷盘（默认） | 最安全，推荐生产用 |
| 2 | 写入 OS cache，由 OS 刷盘 | 介于 0 和 1 之间 |

**建议**：生产环境用 1，别省这点性能。

### binlog 是什么？

redo log 是物理日志，记录"数据页第几行改了什么"。binlog 是逻辑日志，记录"对 user 表 id=1 的行做了更新"。

binlog 在 MySQL Server 层，主要用于：
- 主从复制：从库通过 binlog 同步数据
- 数据恢复：误删数据可以用 binlog 回放

![d7240422b4f8c96201fe28846b49b13e MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/d7240422b4f8c96201fe28846b49b13e_MD5.png)

binlog 的刷盘策略通过 `sync_binlog` 控制：

| 值 | 行为 |
|---|------|
| 0（默认） | 写入 OS cache，由 OS 刷盘 |
| 1 | 每次事务提交都强制落盘 |

**建议**：和 redo log 一样，`sync_binlog = 1` 配合使用，保证 redo log 和 binlog 一致。