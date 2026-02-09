---
{"dg-publish":true,"permalink":"/01-概念卡/数据库/MVCC并发控制机制/","tags":["mvcc","并发控制","事务隔离","概念卡","tech/mysql"]}
---

## 1. 核心概念
MVCC（Multi-Version Concurrency Control）是一种**多版本并发控制**机制，通过保存数据的多个版本，实现**读不加锁、读写不冲突**，大幅提升并发性能。

## 2. 基本原理

### 2.1. 传统锁机制 vs MVCC
```
传统锁机制：
读操作 ──→ 加共享锁 ──→ 阻塞写操作
写操作 ──→ 加排他锁 ──→ 阻塞读写操作

MVCC 机制：
读操作 ──→ 读取历史版本 ──→ 不阻塞写操作
写操作 ──→ 创建新版本 ──→ 不阻塞读操作
```

### 2.2. 核心思想
- 写操作创建新版本，不覆盖旧版本
- 读操作根据事务开始时间读取对应版本
- 通过版本链实现快照读

## 3. 实现机制

### 3.1. 隐藏字段
每行记录包含三个隐藏字段：

```
用户数据 | DB_TRX_ID | DB_ROLL_PTR | DB_ROW_ID
---------|-----------|-------------|----------
id=1     | 100       | 0x1234      | 1
name='张三'
age=25
```

**字段说明**:
- `DB_TRX_ID` (6 bytes): 最后修改该行的事务 ID
- `DB_ROLL_PTR` (7 bytes): 回滚指针，指向 Undo Log 中的历史版本
- `DB_ROW_ID` (6 bytes): 隐藏主键（无主键时使用）

### 3.2. Undo Log 版本链

#### 3.2.1. 版本链示例
```sql
-- 初始数据
INSERT INTO users VALUES (1, '张三', 25);  -- trx_id=100

-- 第一次更新
UPDATE users SET age=26 WHERE id=1;  -- trx_id=101

-- 第二次更新
UPDATE users SET age=27 WHERE id=1;  -- trx_id=102
```

#### 3.2.2. 版本链结构
```
当前版本 (最新)
┌─────────────────────────┐
│ id=1, name='张三', age=27│
│ trx_id=102              │
│ roll_ptr ──────┐        │
└────────────────┼────────┘
                 ↓
版本 2
┌─────────────────────────┐
│ id=1, name='张三', age=26│
│ trx_id=101              │
│ roll_ptr ──────┐        │
└────────────────┼────────┘
                 ↓
版本 1 (最老)
┌─────────────────────────┐
│ id=1, name='张三', age=25│
│ trx_id=100              │
│ roll_ptr=NULL           │
└─────────────────────────┘
```

### 3.3. Read View（读视图）

#### 3.3.1. Read View 结构
```
Read View 包含：
├─ m_ids: [101, 103, 105]     # 活跃事务 ID 列表
├─ min_trx_id: 101             # 最小活跃事务 ID
├─ max_trx_id: 106             # 下一个将分配的事务 ID
└─ creator_trx_id: 102         # 创建 Read View 的事务 ID
```

#### 3.3.2. 可见性判断规则
```
判断 trx_id 是否可见：

1. trx_id < min_trx_id
   → 可见（事务已提交）

2. trx_id >= max_trx_id
   → 不可见（事务未开始）

3. min_trx_id <= trx_id < max_trx_id
   ├─ trx_id == creator_trx_id
   │  → 可见（自己的修改）
   ├─ trx_id in m_ids
   │  → 不可见（活跃事务）
   └─ trx_id not in m_ids
      → 可见（已提交事务）
```

## 4. 隔离级别实现

### 4.1. READ COMMITTED（读已提交）
```sql
-- 每次查询都创建新的 Read View
BEGIN;
SELECT * FROM users WHERE id=1;  -- 创建 Read View 1
-- 其他事务提交
SELECT * FROM users WHERE id=1;  -- 创建 Read View 2（看到新数据）
COMMIT;
```

**特点**: 每次 SELECT 创建新 Read View，可以读到其他事务已提交的数据

### 4.2. REPEATABLE READ（可重复读）
```sql
-- 第一次查询创建 Read View，后续复用
BEGIN;
SELECT * FROM users WHERE id=1;  -- 创建 Read View
-- 其他事务提交
SELECT * FROM users WHERE id=1;  -- 复用 Read View（看不到新数据）
COMMIT;
```

**特点**: 事务开始时创建 Read View，整个事务期间复用，保证可重复读

## 5. 完整示例

### 5.1. 场景：并发更新
```sql
-- 初始数据: id=1, age=25, trx_id=100

-- 事务 A (trx_id=101)
BEGIN;
SELECT age FROM users WHERE id=1;  -- 读取 age=25
-- 创建 Read View: m_ids=[], min=101, max=102

-- 事务 B (trx_id=102)
BEGIN;
UPDATE users SET age=26 WHERE id=1;  -- 创建新版本
COMMIT;

-- 事务 A 再次查询
SELECT age FROM users WHERE id=1;  -- 仍然读取 age=25
-- 原因：trx_id=102 >= max_trx_id=102，不可见
COMMIT;
```

### 5.2. 版本链查找过程
```
1. 读取当前版本: age=26, trx_id=102
   ├─ 判断可见性: 102 >= 102 → 不可见
   └─ 沿着 roll_ptr 查找上一版本

2. 读取版本 1: age=25, trx_id=100
   ├─ 判断可见性: 100 < 101 → 可见
   └─ 返回 age=25
```

## 6. 快照读 vs 当前读

### 6.1. 快照读（Snapshot Read）
```sql
-- 普通 SELECT，使用 MVCC
SELECT * FROM users WHERE id=1;
```

**特点**:
- 读取历史版本
- 不加锁
- 不阻塞其他事务

### 6.2. 当前读（Current Read）
```sql
-- 加锁 SELECT，读取最新版本
SELECT * FROM users WHERE id=1 LOCK IN SHARE MODE;  -- 共享锁
SELECT * FROM users WHERE id=1 FOR UPDATE;          -- 排他锁

-- DML 操作都是当前读
UPDATE users SET age=26 WHERE id=1;
DELETE FROM users WHERE id=1;
INSERT INTO users VALUES (2, '李四', 30);
```

**特点**:
- 读取最新版本
- 加锁
- 可能阻塞其他事务

## 7. 幻读问题

### 7.1. 什么是幻读
```sql
-- 事务 A
BEGIN;
SELECT * FROM users WHERE age > 20;  -- 返回 3 条
-- 事务 B 插入新数据
SELECT * FROM users WHERE age > 20;  -- 返回 4 条（幻读）
COMMIT;
```

### 7.2. MVCC 解决幻读
```sql
-- 可重复读级别
BEGIN;
SELECT * FROM users WHERE age > 20;  -- 快照读，返回 3 条
-- 事务 B 插入新数据并提交
SELECT * FROM users WHERE age > 20;  -- 仍返回 3 条（MVCC）

-- 但当前读会出现幻读
SELECT * FROM users WHERE age > 20 FOR UPDATE;  -- 返回 4 条
```

### 7.3. Next-Key Lock 解决幻读
```sql
-- 当前读 + Next-Key Lock
SELECT * FROM users WHERE age > 20 FOR UPDATE;
-- 锁定范围：(20, +∞)，防止插入新数据
```

## 8. Undo Log 管理

### 8.1. Undo Log 作用
1. **事务回滚**: 撤销未提交的修改
2. **MVCC**: 提供历史版本数据
3. **崩溃恢复**: 恢复未完成的事务

### 8.2. Purge 清理
```
Purge 线程定期清理：
1. 已提交事务的 Undo Log
2. 没有事务需要的历史版本
3. 释放 Undo 表空间
```

### 8.3. 长事务问题
```sql
-- 长事务导致 Undo Log 堆积
BEGIN;
SELECT * FROM users;  -- 创建 Read View
-- 长时间不提交
-- 导致：
-- 1. Undo Log 无法清理
-- 2. 表空间膨胀
-- 3. 性能下降
```

**避免方法**:
- 及时提交事务
- 避免在事务中执行耗时操作
- 监控长事务

## 9. 性能优化

### 9.1. 减少版本链长度
```sql
-- ❌ 频繁更新同一行
UPDATE users SET age=age+1 WHERE id=1;  -- 执行 1000 次

-- ✅ 批量更新
UPDATE users SET age=age+1000 WHERE id=1;  -- 执行 1 次
```

### 9.2. 及时提交事务
```sql
-- ❌ 长事务
BEGIN;
SELECT * FROM users;
-- 执行耗时操作
sleep(3600);
COMMIT;

-- ✅ 短事务
BEGIN;
SELECT * FROM users;
COMMIT;
-- 执行耗时操作
```

### 9.3. 监控 Undo 表空间
```sql
-- 查看 Undo 表空间大小
SELECT 
    tablespace_name,
    file_name,
    ROUND(bytes/1024/1024, 2) AS size_mb
FROM information_schema.files
WHERE tablespace_name LIKE '%undo%';

-- 查看活跃事务
SELECT * FROM information_schema.innodb_trx
WHERE trx_started < DATE_SUB(NOW(), INTERVAL 1 HOUR);
```

## 10. 优缺点

### 10.1. ✅ 优点
1. **高并发**: 读写不冲突
2. **无锁读**: 快照读不加锁
3. **一致性**: 保证事务隔离
4. **性能好**: 减少锁等待

### 10.2. ❌ 缺点
1. **空间开销**: 存储多个版本
2. **版本链**: 过长影响性能
3. **长事务**: 导致 Undo Log 堆积
4. **幻读**: 快照读无法完全避免

## 11. 实战建议

### 11.1. ✅ 推荐做法
1. 使用默认的可重复读隔离级别
2. 及时提交事务，避免长事务
3. 监控 Undo 表空间大小
4. 定期清理历史数据
5. 合理使用当前读和快照读

### 11.2. ❌ 避免做法
1. 在事务中执行耗时操作
2. 频繁更新同一行数据
3. 不监控长事务
4. 滥用 FOR UPDATE
5. 忽略 Undo Log 清理

## 12. 相关概念
- [[01-概念卡/数据库/InnoDB存储引擎原理\|InnoDB存储引擎原理]] - 存储机制
- [[01-概念卡/数据库/MySQL索引原理\|MySQL索引原理]] - 索引结构
- [[01-概念卡/数据库/事务隔离级别\|事务隔离级别]] - 隔离级别详解
- [[01-概念卡/数据库/MySQL锁机制详解\|MySQL锁机制详解]] - 锁的实现

## 13. 参考资源
- [MySQL InnoDB MVCC 官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)

