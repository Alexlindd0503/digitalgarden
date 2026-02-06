---
{"dg-publish":true,"permalink":"/01-概念卡/数据库/InnoDB存储引擎原理/","tags":["innodb","存储引擎","事务","概念卡","tech/mysql"]}
---

## 1. 核心概念
InnoDB 是 MySQL 的默认存储引擎，支持 **ACID 事务**、**行级锁**、**外键约束**和**崩溃恢复**，采用 **MVCC** 实现高并发。

## 2. 架构设计

### 2.1. 整体架构
```
┌─────────────────────────────────────┐
│         InnoDB 架构                  │
├─────────────────────────────────────┤
│  内存结构                            │
│  ├─ Buffer Pool (缓冲池)            │
│  ├─ Change Buffer (写缓冲)          │
│  ├─ Adaptive Hash Index (自适应哈希) │
│  └─ Log Buffer (日志缓冲)           │
├─────────────────────────────────────┤
│  磁盘结构                            │
│  ├─ 表空间 (Tablespace)             │
│  ├─ Redo Log (重做日志)             │
│  ├─ Undo Log (回滚日志)             │
│  └─ Binlog (二进制日志)             │
└─────────────────────────────────────┘
```

## 3. 内存结构

### 3.1. Buffer Pool（缓冲池）

#### 3.1.1. 作用
- 缓存表数据和索引数据
- 减少磁盘 I/O
- 提高读写性能

#### 3.1.2. 结构
```
Buffer Pool
├─ 数据页 (Data Page)
├─ 索引页 (Index Page)
├─ 插入缓冲 (Insert Buffer)
├─ 自适应哈希索引 (Adaptive Hash Index)
└─ 锁信息 (Lock Info)
```

#### 3.1.3. 页管理（LRU 算法）
```
┌──────────────────────────┐
│  New Sublist (5/8)       │ ← 热数据
│  - 最近访问的页          │
├──────────────────────────┤
│  Old Sublist (3/8)       │ ← 冷数据
│  - 预读的页              │
│  - 全表扫描的页          │
└──────────────────────────┘
```

**特点**:
- 新读取的页先放入 Old 区域
- 访问超过 1 秒后移到 New 区域
- 避免全表扫描污染热数据

#### 3.1.4. 配置
```ini
[mysqld]
# 缓冲池大小（物理内存的 70-80%）
innodb_buffer_pool_size = 8G

# 缓冲池实例数（提高并发）
innodb_buffer_pool_instances = 8

# 预读页数
innodb_read_ahead_threshold = 56
```

### 3.2. Change Buffer（写缓冲）

#### 3.2.1. 作用
- 缓存**非唯一二级索引**的修改
- 延迟写入磁盘
- 减少随机 I/O

#### 3.2.2. 工作原理
```sql
-- 插入数据
INSERT INTO users (name, age) VALUES ('张三', 25);

-- 流程
1. 主键索引：直接写入 Buffer Pool
2. 二级索引：写入 Change Buffer（延迟）
3. 后台线程：定期合并到磁盘
```

**适用场景**:
- 写多读少的表
- 非唯一索引
- 大量插入/更新操作

### 3.3. Log Buffer（日志缓冲）

#### 3.3.1. 作用
- 缓存 Redo Log
- 批量写入磁盘
- 提高事务性能

#### 3.3.2. 刷盘策略
```ini
[mysqld]
# 刷盘策略
innodb_flush_log_at_trx_commit = 1

# 0: 每秒刷盘（性能最好，可能丢失 1 秒数据）
# 1: 每次事务提交刷盘（最安全，性能较差）
# 2: 每次提交写入 OS 缓存，每秒刷盘（折中）
```

## 4. 磁盘结构

### 4.1. 表空间（Tablespace）

#### 4.1.1. 系统表空间
```
ibdata1
├─ 数据字典 (Data Dictionary)
├─ 双写缓冲区 (Doublewrite Buffer)
├─ Change Buffer
└─ Undo Log
```

#### 4.1.2. 独立表空间
```
database/
├─ table1.ibd  # 表数据 + 索引
├─ table2.ibd
└─ table3.ibd
```

**配置**:
```ini
[mysqld]
# 每个表独立表空间
innodb_file_per_table = 1
```

#### 4.1.3. 页结构
```
Page (16KB)
├─ File Header (38 bytes)      # 页头
├─ Page Header (56 bytes)      # 页信息
├─ Infimum + Supremum (26 bytes) # 最小/最大记录
├─ User Records                # 用户数据
├─ Free Space                  # 空闲空间
├─ Page Directory              # 页目录
└─ File Trailer (8 bytes)      # 页尾校验
```

### 4.2. Redo Log（重做日志）

#### 4.2.1. 作用
- 保证事务持久性（D）
- 崩溃恢复
- WAL（Write-Ahead Logging）机制

#### 4.2.2. 工作原理
```
事务提交流程：
1. 修改 Buffer Pool 中的数据页
2. 写入 Redo Log Buffer
3. 事务提交时刷盘 Redo Log
4. 后台线程异步刷新数据页

崩溃恢复：
1. 读取 Redo Log
2. 重做已提交事务
3. 回滚未提交事务
```

#### 4.2.3. 循环写入
```
┌─────────────────────────────┐
│  ib_logfile0  ib_logfile1   │
│  ┌──────┐    ┌──────┐       │
│  │ write│───→│      │       │
│  │ pos  │    │      │       │
│  └──────┘    └──────┘       │
│     ↑            │           │
│     └────────────┘           │
│      循环写入                │
└─────────────────────────────┘
```

**配置**:
```ini
[mysqld]
# 日志文件大小
innodb_log_file_size = 512M

# 日志文件数量
innodb_log_files_in_group = 2

# 日志缓冲区大小
innodb_log_buffer_size = 16M
```

### 4.3. Undo Log（回滚日志）

#### 4.3.1. 作用
- 事务回滚
- MVCC 多版本并发控制
- 崩溃恢复

#### 4.3.2. 版本链
```sql
-- 原始数据
id=1, name='张三', age=25, trx_id=100

-- 更新操作
UPDATE users SET age=26 WHERE id=1;  -- trx_id=101
UPDATE users SET age=27 WHERE id=1;  -- trx_id=102

-- Undo Log 版本链
最新版本: age=27, trx_id=102
    ↓
版本 2:   age=26, trx_id=101
    ↓
版本 1:   age=25, trx_id=100
```

#### 4.3.3. Undo 段
```
Undo Tablespace
├─ Rollback Segment 1
│  ├─ Undo Slot 1 (事务 1)
│  ├─ Undo Slot 2 (事务 2)
│  └─ ...
├─ Rollback Segment 2
└─ ...
```

## 5. 事务实现

### 5.1. ACID 特性

#### 5.1.1. 原子性（Atomicity）
**实现**: Undo Log
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- 失败时通过 Undo Log 回滚
ROLLBACK;
```

#### 5.1.2. 一致性（Consistency）
**实现**: Undo Log + Redo Log
- 事务执行前后数据完整性约束保持一致

#### 5.1.3. 隔离性（Isolation）
**实现**: MVCC + 锁机制
```sql
-- 读未提交 (Read Uncommitted)
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- 读已提交 (Read Committed)
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 可重复读 (Repeatable Read) - 默认
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 串行化 (Serializable)
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

#### 5.1.4. 持久性（Durability）
**实现**: Redo Log + Doublewrite Buffer
- 事务提交后数据永久保存

### 5.2. MVCC 机制

#### 5.2.1. 隐藏字段
```
每行记录包含：
├─ DB_TRX_ID (6 bytes)    # 最后修改的事务 ID
├─ DB_ROLL_PTR (7 bytes)  # 回滚指针，指向 Undo Log
└─ DB_ROW_ID (6 bytes)    # 隐藏主键（无主键时）
```

#### 5.2.2. Read View
```
Read View 包含：
├─ trx_ids          # 活跃事务列表
├─ low_limit_id     # 最大事务 ID + 1
├─ up_limit_id      # 最小活跃事务 ID
└─ creator_trx_id   # 创建 Read View 的事务 ID
```

#### 5.2.3. 可见性判断
```
判断规则：
1. trx_id < up_limit_id        → 可见（已提交）
2. trx_id >= low_limit_id      → 不可见（未开始）
3. trx_id in trx_ids           → 不可见（活跃中）
4. trx_id == creator_trx_id    → 可见（自己的修改）
5. 其他                        → 可见（已提交）
```

#### 5.2.4. 示例
```sql
-- 事务 A (trx_id=100)
BEGIN;
SELECT * FROM users WHERE id = 1;  -- age=25

-- 事务 B (trx_id=101)
BEGIN;
UPDATE users SET age=26 WHERE id = 1;
COMMIT;

-- 事务 A 再次查询
SELECT * FROM users WHERE id = 1;  -- age=25 (可重复读)

-- 原理：
-- 事务 A 的 Read View: trx_ids=[100], up_limit_id=100
-- 版本链中 trx_id=101 > up_limit_id，不可见
-- 通过 Undo Log 读取 trx_id=100 的版本
```

## 6. 锁机制

### 6.1. 锁类型

#### 6.1.1. 共享锁（S Lock）
```sql
-- 读锁，允许多个事务同时读
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;
```

#### 6.1.2. 排他锁（X Lock）
```sql
-- 写锁，独占访问
SELECT * FROM users WHERE id = 1 FOR UPDATE;
UPDATE users SET age = 26 WHERE id = 1;
```

#### 6.1.3. 意向锁（Intention Lock）
- **IS**: 意向共享锁
- **IX**: 意向排他锁
- 表级锁，用于提高加表锁的效率

### 6.2. 锁粒度

#### 6.2.1. 行锁（Row Lock）
```sql
-- 锁定单行
UPDATE users SET age = 26 WHERE id = 1;
```

#### 6.2.2. 间隙锁（Gap Lock）
```sql
-- 锁定范围（防止幻读）
SELECT * FROM users WHERE id BETWEEN 10 AND 20 FOR UPDATE;
-- 锁定 (10, 20) 之间的间隙
```

#### 6.2.3. Next-Key Lock
```
Next-Key Lock = Record Lock + Gap Lock

示例：索引值 [10, 20, 30]
锁定 id=20:
├─ Record Lock: 锁定 20
└─ Gap Lock: 锁定 (10, 20]
```

### 6.3. 死锁

#### 6.3.1. 死锁示例
```sql
-- 事务 A
BEGIN;
UPDATE users SET age = 26 WHERE id = 1;  -- 锁定 id=1
UPDATE users SET age = 27 WHERE id = 2;  -- 等待 id=2

-- 事务 B
BEGIN;
UPDATE users SET age = 28 WHERE id = 2;  -- 锁定 id=2
UPDATE users SET age = 29 WHERE id = 1;  -- 等待 id=1（死锁）
```

#### 6.3.2. 死锁检测
```sql
-- 查看死锁日志
SHOW ENGINE INNODB STATUS;

-- 死锁检测配置
SET GLOBAL innodb_deadlock_detect = ON;
```

#### 6.3.3. 避免死锁
1. 按相同顺序访问资源
2. 缩短事务时间
3. 降低隔离级别
4. 使用乐观锁

## 7. 崩溃恢复

### 7.1. 恢复流程
```
1. 读取 Redo Log
   ↓
2. 重做已提交事务（前滚）
   ↓
3. 读取 Undo Log
   ↓
4. 回滚未提交事务（回滚）
   ↓
5. 恢复完成
```

### 7.2. Checkpoint 机制
```
Checkpoint 作用：
├─ 缩短恢复时间
├─ 刷新脏页到磁盘
└─ 释放 Redo Log 空间

触发时机：
├─ Master Thread 定期触发
├─ Redo Log 空间不足
├─ 脏页比例过高
└─ 数据库关闭
```

### 7.3. Doublewrite Buffer
```
写入流程：
1. 脏页写入 Doublewrite Buffer (内存)
   ↓
2. Doublewrite Buffer 刷盘（顺序写）
   ↓
3. 脏页写入表空间（随机写）

作用：
- 防止部分写入（Partial Write）
- 保证数据页完整性
```

## 8. 性能优化

### 8.1. Buffer Pool 优化
```sql
-- 查看缓冲池状态
SHOW ENGINE INNODB STATUS;

-- 查看缓冲池命中率
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';

-- 计算命中率
命中率 = (1 - Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests) * 100%
-- 目标：> 99%
```

### 8.2. 日志优化
```ini
[mysqld]
# 增大日志文件（减少 Checkpoint 频率）
innodb_log_file_size = 1G

# 调整刷盘策略（性能 vs 安全）
innodb_flush_log_at_trx_commit = 2
```

### 8.3. I/O 优化
```ini
[mysqld]
# 增加 I/O 线程
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# 刷新邻接页（SSD 建议关闭）
innodb_flush_neighbors = 0
```

## 9. 监控指标

### 9.1. 关键指标
```sql
-- 缓冲池使用率
SELECT 
    (1 - (SELECT VARIABLE_VALUE FROM performance_schema.global_status 
          WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
         (SELECT VARIABLE_VALUE FROM performance_schema.global_status 
          WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')) * 100 
    AS buffer_pool_hit_rate;

-- 脏页比例
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_dirty';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_total';

-- 锁等待
SELECT * FROM information_schema.innodb_lock_waits;

-- 事务状态
SELECT * FROM information_schema.innodb_trx;
```

## 10. 与 MyISAM 对比

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 支持 | ❌ 不支持 |
| 行级锁 | ✅ 支持 | ❌ 表级锁 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| MVCC | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复 | ❌ 需要修复 |
| 全文索引 | ✅ 5.6+ | ✅ 支持 |
| 适用场景 | 高并发、事务 | 读多写少 |

## 11. 关键要点

1. **Buffer Pool** 是性能关键，建议设置为物理内存的 70-80%
2. **Redo Log** 保证持久性，采用 WAL 机制
3. **Undo Log** 实现事务回滚和 MVCC
4. **MVCC** 通过版本链实现非锁定读
5. **行锁 + MVCC** 实现高并发
6. **Checkpoint** 机制平衡性能和恢复时间
7. **Doublewrite Buffer** 防止数据页损坏

## 12. 相关概念
- [[03-方法卡/数据库/MySQL性能优化\|MySQL性能优化]] - 性能调优
- [[03-方法卡/数据库/MySQL大表数据删除方案\|MySQL大表数据删除方案]] - 数据管理
- [[03-方法卡/数据库/MySQL备份恢复方案\|MySQL备份恢复方案]] - 数据安全
- [[01-概念卡/数据库/MVCC并发控制机制\|MVCC并发控制机制]] - 并发原理
- [[01-概念卡/数据库/MySQL索引原理\|MySQL索引原理]] - 索引实现

## 13. 参考资源
- [MySQL InnoDB 官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)
- [高性能 MySQL](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)