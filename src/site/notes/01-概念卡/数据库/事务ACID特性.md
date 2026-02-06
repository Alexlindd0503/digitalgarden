---
{"dg-publish":true,"permalink":"/01-概念卡/数据库/事务ACID特性/","tags":["事务","ACID","数据一致性","概念卡","tech/mysql"]}
---

## 1. 核心概念
ACID 是事务的四大特性，保证数据库操作的**可靠性**和**一致性**。ACID 是 Atomicity（原子性）、Consistency（一致性）、Isolation（隔离性）、Durability（持久性）的缩写。

## 2. 四大特性

### 2.1. A - 原子性（Atomicity）

#### 2.1.1. 定义
事务是**不可分割**的最小工作单元，要么全部成功，要么全部失败回滚。

#### 2.1.2. 实现机制
**Undo Log（回滚日志）**

```sql
-- 转账事务
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 扣款
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 加款
-- 如果失败，通过 Undo Log 回滚
ROLLBACK;
```

#### 2.1.3. 工作原理
```
1. 事务开始前，记录原始数据到 Undo Log
   ├─ 修改前: balance=1000
   └─ 写入 Undo Log

2. 执行修改操作
   └─ 修改后: balance=900

3. 事务回滚时
   └─ 从 Undo Log 读取原始值，恢复数据
```

#### 2.1.4. 示例
```sql
-- 原子性保证
BEGIN;
INSERT INTO orders (id, amount) VALUES (1, 100);
INSERT INTO order_items (order_id, product_id) VALUES (1, 101);
-- 第二条 SQL 失败，第一条也会回滚
ROLLBACK;
```

### 2.2. C - 一致性（Consistency）

#### 2.2.1. 定义
事务执行前后，数据库从一个**一致性状态**转换到另一个**一致性状态**，满足所有完整性约束。

#### 2.2.2. 实现机制
**Undo Log + Redo Log + 约束检查**

#### 2.2.3. 一致性约束
```sql
-- 1. 主键约束
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);

-- 2. 外键约束
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- 3. 唯一约束
CREATE TABLE users (
    email VARCHAR(100) UNIQUE
);

-- 4. 检查约束
CREATE TABLE products (
    price DECIMAL(10,2) CHECK (price > 0)
);
```

#### 2.2.4. 示例：转账一致性
```sql
-- 转账前: A=1000, B=500, 总额=1500
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';  -- A=900
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';  -- B=600
COMMIT;
-- 转账后: A=900, B=600, 总额=1500（一致）
```

### 2.3. I - 隔离性（Isolation）

#### 2.3.1. 定义
多个事务并发执行时，一个事务的执行**不应影响**其他事务，每个事务都感觉像是独占数据库。

#### 2.3.2. 实现机制
**MVCC + 锁机制**

#### 2.3.3. 隔离级别
```sql
-- 读未提交（最低隔离）
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- 读已提交
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 可重复读（MySQL 默认）
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 串行化（最高隔离）
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

#### 2.3.4. 示例：隔离性保证
```sql
-- 事务 A
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 1000
-- 事务 B 修改并提交
SELECT balance FROM accounts WHERE id = 1;  -- 仍然 1000（可重复读）
COMMIT;
```

### 2.4. D - 持久性（Durability）

#### 2.4.1. 定义
事务一旦提交，对数据库的修改就是**永久性**的，即使系统崩溃也不会丢失。

#### 2.4.2. 实现机制
**Redo Log（重做日志）+ Doublewrite Buffer**

#### 2.4.3. 工作原理
```
1. 事务修改数据
   └─ 修改 Buffer Pool 中的数据页

2. 写入 Redo Log
   └─ 记录数据页的修改（WAL 机制）

3. 事务提交
   └─ Redo Log 刷盘（持久化）

4. 后台异步刷新数据页
   └─ 将脏页写入磁盘

5. 崩溃恢复
   └─ 通过 Redo Log 重做已提交事务
```

#### 2.4.4. WAL（Write-Ahead Logging）
```
先写日志，再写数据

优势：
├─ 顺序写日志（快）
├─ 随机写数据（慢，异步）
└─ 保证持久性
```

#### 2.4.5. 示例
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;  -- Redo Log 刷盘，保证持久性

-- 即使此时数据库崩溃，重启后也能恢复
```

## 3. ACID 实现总结

### 3.1. 实现机制对应表
| 特性 | 实现机制 | 关键组件 |
|------|---------|----------|
| 原子性 | Undo Log | 回滚日志 |
| 一致性 | Undo Log + Redo Log + 约束 | 日志 + 约束检查 |
| 隔离性 | MVCC + 锁 | Read View + 锁机制 |
| 持久性 | Redo Log + Doublewrite | 重做日志 + 双写缓冲 |

### 3.2. 日志作用
```
Undo Log:
├─ 事务回滚（原子性）
├─ MVCC 多版本（隔离性）
└─ 崩溃恢复（一致性）

Redo Log:
├─ 事务持久化（持久性）
├─ 崩溃恢复（一致性）
└─ WAL 机制（性能）
```

## 4. 完整示例

### 4.1. 转账事务
```sql
-- 转账 100 元从账户 A 到账户 B
BEGIN;

-- 1. 检查余额（一致性）
SELECT balance FROM accounts WHERE id = 'A' FOR UPDATE;
-- balance = 1000

-- 2. 扣款（原子性 - 记录 Undo Log）
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
-- Undo Log: A balance 1000 → 900

-- 3. 加款（原子性 - 记录 Undo Log）
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
-- Undo Log: B balance 500 → 600

-- 4. 提交（持久性 - Redo Log 刷盘）
COMMIT;
-- Redo Log 持久化，保证数据不丢失

-- 隔离性：其他事务看不到中间状态
-- 一致性：总金额保持不变（1500）
```

### 4.2. 崩溃恢复
```
场景：事务提交后，数据页未刷盘，数据库崩溃

恢复流程：
1. 读取 Redo Log
2. 重做已提交事务（持久性）
3. 读取 Undo Log
4. 回滚未提交事务（原子性）
5. 恢复完成（一致性）
```

## 5. 配置优化

### 5.1. Redo Log 配置
```ini
[mysqld]
# Redo Log 文件大小
innodb_log_file_size = 512M

# Redo Log 缓冲区
innodb_log_buffer_size = 16M

# 刷盘策略
innodb_flush_log_at_trx_commit = 1
# 0: 每秒刷盘（性能最好，可能丢失 1 秒数据）
# 1: 每次提交刷盘（最安全，性能较差）
# 2: 每次提交写 OS 缓存，每秒刷盘（折中）
```

### 5.2. Undo Log 配置
```ini
[mysqld]
# Undo 表空间数量
innodb_undo_tablespaces = 2

# Undo 日志回收
innodb_undo_log_truncate = ON
innodb_max_undo_log_size = 1G
```

## 6. 性能权衡

### 6.1. 安全性 vs 性能
```
innodb_flush_log_at_trx_commit = 1
├─ 安全性: ⭐⭐⭐⭐⭐（最高）
├─ 性能:   ⭐⭐（较低）
└─ 数据丢失: 0

innodb_flush_log_at_trx_commit = 2
├─ 安全性: ⭐⭐⭐⭐（高）
├─ 性能:   ⭐⭐⭐⭐（较高）
└─ 数据丢失: OS 崩溃时丢失

innodb_flush_log_at_trx_commit = 0
├─ 安全性: ⭐⭐⭐（中）
├─ 性能:   ⭐⭐⭐⭐⭐（最高）
└─ 数据丢失: MySQL 崩溃时丢失 1 秒
```

## 7. 常见问题

### 7.1. Q1: 为什么需要 Undo Log 和 Redo Log 两种日志？
```
Undo Log:
- 用于回滚（原子性）
- 用于 MVCC（隔离性）
- 记录修改前的数据

Redo Log:
- 用于持久化（持久性）
- 用于崩溃恢复
- 记录修改后的数据

两者配合保证 ACID
```

### 7.2. Q2: 事务提交后数据一定在磁盘上吗？
```
不一定：
1. Redo Log 一定在磁盘（持久性保证）
2. 数据页可能在 Buffer Pool（异步刷盘）
3. 崩溃后通过 Redo Log 恢复数据页
```

### 7.3. Q3: 如何保证转账不会出现不一致？
```
1. 原子性: 要么都成功，要么都失败
2. 一致性: 总金额保持不变
3. 隔离性: 其他事务看不到中间状态
4. 持久性: 提交后永久保存
```

## 8. 最佳实践

### 8.1. ✅ 推荐做法
1. 使用事务保证数据一致性
2. 合理设置隔离级别
3. 及时提交事务，避免长事务
4. 生产环境使用 `innodb_flush_log_at_trx_commit=1`
5. 定期监控 Undo Log 大小

### 8.2. ❌ 避免做法
1. 不使用事务处理关键业务
2. 在事务中执行耗时操作
3. 为了性能牺牲数据安全
4. 忽略事务隔离级别
5. 不监控长事务

## 9. 相关概念
- [[01-概念卡/数据库/InnoDB存储引擎原理\|InnoDB存储引擎原理]] - 存储机制
- [[01-概念卡/数据库/MVCC并发控制机制\|MVCC并发控制机制]] - 隔离性实现
- [[01-概念卡/数据库/事务隔离级别\|事务隔离级别]] - 隔离级别详解
- [[01-概念卡/数据库/MySQL锁机制详解\|MySQL锁机制详解]] - 锁机制

## 10. 参考资源
- [MySQL 事务官方文档](https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html)
