---
{"dg-publish":true,"permalink":"/01.专项学习/MySQL实战高手/05-事务、MVCC与锁/","dg-note-properties":{"时间":"2026-03-22"}}
---

#mysql #数据库 #事务 #MVCC

```ad-summary
title: 总结

- 并发事务四大问题：脏写/脏读（读到未提交）、不可重复读（同一条数据变了）、幻读（多出几条数据）
- MySQL 默认 RR（可重复读）级别，且能避免幻读，生产环境基本够用
- MVCC 核心：undo log 版本链 + ReadView，让读操作不用加锁
- RC 每次查询重新生成 ReadView，RR 事务内只生成一次，这就是区别
- 行锁（独占锁/共享锁）比表锁实用，但实际开发更常用分布式锁控制业务逻辑
```

## 1. 并发事务会出什么问题？

多个事务同时操作同一批数据，会有四种典型问题：

### 1.1 脏写

A 更新值为 A，B 接着更新值为 B，A 回滚了，B 的更新也被"吃掉"了。

场景：A、B 都更新同一行。A 先改值为 A，B 再改值为 B，A 回滚把值恢复成 NULL。B 明明更新成功了，结果却没了，这就是脏写。

### 1.2 脏读

A 更新了数据，B 读到了 A 更新的值，但 A 回滚了，B 读到的值就是脏的。

**脏写和脏读的本质**：一个事务操作了另一个还没提交的事务的数据。因为对方随时可能回滚，你读到/写入的值就不可靠。

### 1.3 不可重复读

A 事务内第一次读是值 X，第二次读变成了值 Y（被 B 改了）。同一个事务里，同样的查询，结果不一致。

![8397cc842177c60e61e104ee68d5bcc1 MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/8397cc842177c60e61e104ee68d5bcc1_MD5.png)

### 1.4 幻读

A 事务内用同样的 SQL 查询，第一次查出 5 条，第二次查出 6 条，多出来的就是"幻影"。

![24375fb4d756aca7a472a742c060efcf MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/24375fb4d756aca7a472a742c060efcf_MD5.png)

**和不可重复读的区别：不可重复读是同一行数据变了，幻读是多出或少了行。**

## 2. 事务隔离级别

SQL 标准定义了 4 种隔离级别，MySQL 都支持：

| 隔离级别             | 脏读  | 不可重复读 | 幻读  | 说明       |
| ---------------- | --- | ----- | --- | -------- |
| READ UNCOMMITTED | ✓   | ✓     | ✓   | 最弱，基本没人用 |
| READ COMMITTED   | ✗   | ✓     | ✓   | 读已提交     |
| REPEATABLE READ  | ✗   | ✗     | ✓   | MySQL 默认 |
| SERIALIZABLE     | ✗   | ✗     | ✗   | 最强，性能最差  |

**MySQL 默认 RR 级别**，而且 MySQL 的 RR 是能避免幻读的（靠 MVCC + 间隙锁），比 SQL 标准规定的更强。

## 3. RC 和 RR 怎么选？

Spring 里通过 `@Transactional(isolation = Isolation.REPEATABLE_READ)` 可以设置，一般保持默认就行。但有些场景可以考虑降到 RC。

### 3.1 RC 的优势：没有间隙锁

RR 级别下，为了防止幻读，会加**间隙锁**（Gap Lock）和**临键锁**（Next-Key Lock）。比如你 `SELECT * FROM user WHERE age > 20 FOR UPDATE`，RR 会锁住 age > 20 的整个间隙，其他事务在这个范围内插入数据会被阻塞。

RC 纺别下没有间隙锁，只有行锁，高并发写入时锁竞争会小很多。

### 3.2 性能差异

- **高并发写入场景**：RC 比 RR 快 10%~30%（减少锁竞争）
- **读多写少场景**：差异不大，RR 够用

### 3.3 什么时候用 RC？

- 业务上能接受不可重复读（比如实时库存、排行榜，查两次不一样没关系）
- 高并发写入，想减少锁竞争
- 不依赖间隙锁来保证业务逻辑

### 3.4 什么时候坚持 RR？

- 需要事务内多次查询结果一致（比如财务对账、订单状态判断）
- 业务逻辑依赖可重复读的语义

### 3.5 同一条 SQL，RR 和 RC 加的锁不一样

假设表里 age 字段有值：10、20、30、40，执行：

```sql
SELECT * FROM user WHERE age > 20 FOR UPDATE;
```

**RR 级别**：加临键锁（Next-Key Lock = 记录锁 + 间隙锁）
- 锁住记录：30、40
- 锁住间隙：(20, 30]、(30, 40]、(40, +∞)
- 其他事务想插 age=25 或 age=50，都会被阻塞

**RC 级别**：只加记录锁（行锁）
- 锁住记录：30、40
- 间隙不锁，其他事务插 age=25 或 age=50 没问题

所以 RC 下锁的范围小得多，高并发插入时不会互相阻塞，性能自然好。

## 4. MVCC 机制

读写操作不想互相阻塞，就要靠 MVCC（Multi-Version Concurrency Control）。

### 4.1 undo log 版本链

每条数据有两个隐藏字段：

- `trx_id`：最后修改这条数据的事务 ID
- `roll_pointer`：指向修改前生成的 undo log

每次更新都会生成一个 undo log，undo log 之间用 roll_pointer 串成一条链，这就是**版本链**。

![d131b484772208cdadbe2b1ad7debef3 MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/d131b484772208cdadbe2b1ad7debef3_MD5.png)

### 4.2 ReadView 机制

事务执行查询时，会生成一个 ReadView，包含 4 个关键信息：

| 字段             | 说明                              |
| -------------- | ------------------------------- |
| m_ids          | 生成 ReadView 时，还在执行未提交的事务 ID 列表  |
| min_trx_id     | m_ids 中最小的事务 ID                 |
| max_trx_id     | MySQL 下一个要生成的事务 ID（当前最大 ID + 1） |
| creator_trx_id | 当前事务自己的 ID                      |

![025abf0f82a80bfa3de13c732ba5d86b MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/025abf0f82a80bfa3de13c732ba5d86b_MD5.png)

#### 事务 ID 怎么保证递增的？

ReadView 的可见性判断依赖 `trx_id < min_trx_id` 这样的比较，前提是事务 ID 必须递增。

MySQL 内部有一个**全局事务 ID 计数器**，每次开启一个新事务（涉及写操作）时，从这个计数器里取一个递增的 ID。计数器是单调递增的，保证了事务 ID 的顺序。

注意：**只读事务不会分配事务 ID**。只有执行 INSERT、UPDATE、DELETE 等写操作时，才会分配事务 ID。所以一个只读事务的 `creator_trx_id` 是 0。

#### 可见性判断逻辑

沿着版本链从新到旧找：

1. 如果版本的 `trx_id < min_trx_id`：说明这个版本在 ReadView 之前就提交了，**可见**
2. 如果版本的 `trx_id >= max_trx_id`：说明这个版本在 ReadView 之后才产生，**不可见**
3. 如果 `min_trx_id <= trx_id < max_trx_id`：
   - `trx_id` 在 m_ids 中：说明该事务还没提交，**不可见**
   - `trx_id` 不在 m_ids 中：说明该事务已提交，**可见**
4. 如果版本的 `trx_id == creator_trx_id`：是自己改的，**可见**

### 4.3 RC 和 RR 的区别

**RC（READ COMMITTED）**：每次查询都重新生成 ReadView

所以 B 提交后，A 再次查询时 ReadView 里已经没有 B 了，就能读到 B 的修改。

![2e3877d70f40ac7cd4e8670c35c7d89e MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/2e3877d70f40ac7cd4e8670c35c7d89e_MD5.png)

**RR（REPEATABLE READ）**：事务内只在第一次查询时生成 ReadView，后续复用

所以即使 B 提交了，A 的 ReadView 还是旧的，读不到 B 的修改，保证可重复读。

![3a4261a969b5d20584f5018b660d7599 MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/3a4261a969b5d20584f5018b660d7599_MD5.png)

幻读的处理也是一样的原理：A 事务内看不到其他事务新插入的行。

## 5. 行锁

### 5.1 快照读和当前读

普通 SELECT 不加锁，是**快照读**，通过 MVCC 读 undo log 版本链：

```sql
-- 快照读，不加锁
SELECT * FROM user WHERE age > 20;
```

只有**当前读**才加锁：

```sql
-- 当前读，加独占锁
SELECT * FROM user WHERE age > 20 FOR UPDATE;

-- 当前读，加共享锁
SELECT * FROM user WHERE age > 20 LOCK IN SHARE MODE;

-- INSERT、UPDATE、DELETE 也是当前读，会加锁
```

**区别**：快照读读的是历史版本（undo log），不阻塞别人也不被别人阻塞；当前读读的是最新版本，需要加锁保证数据不被并发修改。

### 5.2 共享锁和独占锁

```sql
-- 共享锁（S Lock）：允许多个事务同时读，但不让别人写
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;

-- 独占锁（X Lock）：我占着，别人读写都不行
SELECT * FROM user WHERE id = 1 FOR UPDATE;
```

锁的互斥关系：

|     | 共享锁        | 独占锁 |
| --- | ---------- | --- |
| 共享锁 | 不互斥（可以同时加） | 互斥  |
| 独占锁 | 互斥         | 互斥  |

#### 什么场景加什么锁？

**加共享锁**：读的时候防止数据被改

```sql
-- 转账前查余额，查的过程中不让别人改余额
BEGIN;
SELECT balance FROM account WHERE id = 1 LOCK IN SHARE MODE;
-- 拿到余额后做业务判断
-- ...
COMMIT;
```

**加独占锁**：改数据时防止别人读写

```sql
-- 扣库存，必须独占，防止超卖
BEGIN;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

**总结**：共享锁用在"我要读，但不想读的过程中被别人改"；独占锁用在"我要改，改之前先锁住防并发"。实际开发中独占锁用得更多，共享锁场景比较少。

### 5.3 记录锁、间隙锁、临键锁

InnoDB 的行锁细分下来有三种，锁的范围不同：

假设表里 age 字段有值：10、20、30、40

**记录锁（Record Lock）**：锁住单条索引记录

```sql
SELECT * FROM user WHERE age = 30 FOR UPDATE;
```
- 只锁住 age=30 这一条记录

**间隙锁（Gap Lock）**：锁住两条记录之间的间隙，不包括记录本身

```sql
SELECT * FROM user WHERE age = 25 FOR UPDATE;
```
- age=25 不存在，锁住 (20, 30) 这个间隙
- 其他事务不能在这个间隙里插入数据

**临键锁（Next-Key Lock）**：记录锁 + 间隙锁，锁住记录和它前面的间隙

```sql
SELECT * FROM user WHERE age > 20 FOR UPDATE;
```
- 锁住 (20, 30]、(30, 40]、(40, +∞)
- 既锁住记录 30、40，也锁住它们前面的间隙

**总结**：RR 级别下会用临键锁防止幻读，RC 级别下只有记录锁，没有间隙锁。这就是为什么 RC 高并发写入性能更好。

### 5.4 表锁和意向锁

MySQL 的表锁其实很鸡肋，基本没人手动加：

```sql
LOCK TABLES xxx READ;   -- 表级共享锁
LOCK TABLES xxx WRITE;  -- 表级独占锁
```

平时说的"表锁"其实是**元数据锁**（Metadata Lock），DDL 操作时自动加的，防止改表结构和改数据冲突。

InnoDB 还有**意向锁**：
- 行级加独占锁时，表级自动加**意向独占锁**
- 行级加共享锁时，表级自动加**意向共享锁**

意向锁之间不互斥，只是用来快速判断"这张表里有没有行被锁了"，不用一行行去检查。
