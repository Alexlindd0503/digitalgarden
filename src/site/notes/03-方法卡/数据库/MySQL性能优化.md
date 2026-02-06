---
{"dg-publish":true,"permalink":"/03-方法卡/数据库/MySQL性能优化/","tags":["性能优化","索引优化","SQL调优","方法卡","tech/mysql"]}
---

## 1. 优化层次

```
应用层优化
    ↓
SQL 查询优化
    ↓
索引优化
    ↓
表结构优化
    ↓
配置参数优化
    ↓
硬件优化
```

## 2. 索引优化

### 2.1. 索引设计原则

#### 2.1.1. 选择性原则
```sql
-- 计算列的选择性（越接近 1 越好）
SELECT 
    COUNT(DISTINCT column_name) / COUNT(*) AS selectivity
FROM table_name;

-- 选择性高的列适合建索引
-- 选择性 > 0.1 建议建索引
```

#### 2.1.2. 最左前缀原则
```sql
-- 联合索引 (a, b, c)
CREATE INDEX idx_abc ON orders(user_id, status, create_time);

-- ✅ 可以使用索引
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 1 AND status = 1;
SELECT * FROM orders WHERE user_id = 1 AND status = 1 AND create_time > '2024-01-01';

-- ❌ 无法使用索引
SELECT * FROM orders WHERE status = 1;
SELECT * FROM orders WHERE create_time > '2024-01-01';
```

#### 2.1.3. 覆盖索引
```sql
-- 查询的列都在索引中，无需回表
CREATE INDEX idx_user_status ON orders(user_id, status);

-- ✅ 覆盖索引，性能最优
SELECT user_id, status FROM orders WHERE user_id = 1;

-- ❌ 需要回表查询其他列
SELECT * FROM orders WHERE user_id = 1;
```

### 2.2. 索引失效场景

#### 2.2.1. 函数操作
```sql
-- ❌ 索引失效
SELECT * FROM orders WHERE DATE(create_time) = '2024-01-01';
SELECT * FROM orders WHERE YEAR(create_time) = 2024;

-- ✅ 使用索引
SELECT * FROM orders 
WHERE create_time >= '2024-01-01' 
  AND create_time < '2024-01-02';
```

#### 2.2.2. 隐式类型转换
```sql
-- order_no 是 VARCHAR 类型
-- ❌ 索引失效（字符串转数字）
SELECT * FROM orders WHERE order_no = 123456;

-- ✅ 使用索引
SELECT * FROM orders WHERE order_no = '123456';
```

#### 2.2.3. 前导模糊查询
```sql
-- ❌ 索引失效
SELECT * FROM users WHERE name LIKE '%张%';
SELECT * FROM users WHERE name LIKE '%张';

-- ✅ 使用索引
SELECT * FROM users WHERE name LIKE '张%';
```

#### 2.2.4. OR 条件
```sql
-- ❌ 可能索引失效
SELECT * FROM orders WHERE user_id = 1 OR status = 1;

-- ✅ 改用 UNION
SELECT * FROM orders WHERE user_id = 1
UNION
SELECT * FROM orders WHERE status = 1;
```

#### 2.2.5. 不等于操作
```sql
-- ❌ 索引失效
SELECT * FROM orders WHERE status != 1;
SELECT * FROM orders WHERE status <> 1;

-- ✅ 使用 IN
SELECT * FROM orders WHERE status IN (0, 2, 3);
```

### 2.3. 索引监控
```sql
-- 查看索引使用情况
SELECT 
    table_schema,
    table_name,
    index_name,
    cardinality,
    seq_in_index
FROM information_schema.statistics
WHERE table_schema = 'mydb'
ORDER BY table_name, index_name;

-- 查看未使用的索引
SELECT 
    object_schema,
    object_name,
    index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND count_star = 0
  AND object_schema = 'mydb';

-- 查看索引大小
SELECT 
    table_name,
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE database_name = 'mydb'
  AND stat_name = 'size';
```

## 3. SQL 查询优化

### 3.1. EXPLAIN 分析

#### 3.1.1. 关键字段解读
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1;
```

| 字段 | 说明 | 优化目标 |
|------|------|----------|
| type | 访问类型 | system > const > eq_ref > ref > range > index > ALL |
| key | 使用的索引 | 确保使用了合适的索引 |
| rows | 扫描行数 | 越少越好 |
| Extra | 额外信息 | 避免 Using filesort, Using temporary |

#### 3.1.2. 优化示例
```sql
-- ❌ 全表扫描
EXPLAIN SELECT * FROM orders WHERE status = 1;
-- type: ALL, rows: 1000000

-- ✅ 添加索引
CREATE INDEX idx_status ON orders(status);
EXPLAIN SELECT * FROM orders WHERE status = 1;
-- type: ref, rows: 10000
```

### 3.2. 分页优化

#### 3.2.1. 深度分页问题
```sql
-- ❌ 性能差（扫描 + 丢弃大量数据）
SELECT * FROM orders 
ORDER BY id 
LIMIT 1000000, 20;

-- ✅ 使用子查询优化
SELECT * FROM orders 
WHERE id >= (
    SELECT id FROM orders 
    ORDER BY id 
    LIMIT 1000000, 1
)
ORDER BY id 
LIMIT 20;

-- ✅ 使用延迟关联
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders 
    ORDER BY id 
    LIMIT 1000000, 20
) AS t ON o.id = t.id;

-- ✅ 记录上次位置
SELECT * FROM orders 
WHERE id > 1000000 
ORDER BY id 
LIMIT 20;
```

### 3.3. JOIN 优化

#### 3.3.1. 小表驱动大表
```sql
-- ❌ 大表驱动小表
SELECT * FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE u.status = 1;

-- ✅ 小表驱动大表
SELECT * FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 1;
```

#### 3.3.2. 避免笛卡尔积
```sql
-- ❌ 笛卡尔积
SELECT * FROM orders, users;

-- ✅ 明确 JOIN 条件
SELECT * FROM orders o
INNER JOIN users u ON o.user_id = u.id;
```

### 3.4. 子查询优化
```sql
-- ❌ 相关子查询（每行都执行）
SELECT * FROM orders o
WHERE o.user_id IN (
    SELECT id FROM users WHERE status = 1
);

-- ✅ 改用 JOIN
SELECT o.* FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE u.status = 1;

-- ✅ 使用 EXISTS（适合小结果集）
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM users u 
    WHERE u.id = o.user_id AND u.status = 1
);
```

### 3.5. COUNT 优化
```sql
-- ❌ 慢（全表扫描）
SELECT COUNT(*) FROM orders;

-- ✅ 使用覆盖索引
SELECT COUNT(id) FROM orders;

-- ✅ 使用近似值（大表）
SELECT table_rows FROM information_schema.tables
WHERE table_schema = 'mydb' AND table_name = 'orders';

-- ✅ 维护计数表
CREATE TABLE order_count (
    date DATE PRIMARY KEY,
    count INT
);
```

## 4. 表结构优化

### 4.1. 字段类型选择

#### 4.1.1. 数值类型
```sql
-- ❌ 浪费空间
user_id BIGINT  -- 实际只需要 INT

-- ✅ 合适的类型
user_id INT UNSIGNED  -- 0 到 42 亿
status TINYINT        -- -128 到 127
price DECIMAL(10,2)   -- 精确金额
```

#### 4.1.2. 字符串类型
```sql
-- ❌ 固定长度浪费空间
name VARCHAR(255)  -- 实际最长 50

-- ✅ 合适的长度
name VARCHAR(50)
email VARCHAR(100)

-- ✅ 固定长度用 CHAR
country_code CHAR(2)  -- CN, US
```

#### 4.1.3. 时间类型
```sql
-- ❌ 使用字符串存储
create_time VARCHAR(20)

-- ✅ 使用 DATETIME/TIMESTAMP
create_time DATETIME DEFAULT CURRENT_TIMESTAMP
update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

### 4.2. 表拆分

#### 4.2.1. 垂直拆分
```sql
-- ❌ 大字段影响查询
CREATE TABLE articles (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,  -- 大字段
    author_id INT,
    create_time DATETIME
);

-- ✅ 拆分大字段
CREATE TABLE articles (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    author_id INT,
    create_time DATETIME
);

CREATE TABLE article_content (
    article_id INT PRIMARY KEY,
    content TEXT,
    FOREIGN KEY (article_id) REFERENCES articles(id)
);
```

#### 4.2.2. 水平拆分
```sql
-- 按时间分表
CREATE TABLE orders_2024_01 LIKE orders;
CREATE TABLE orders_2024_02 LIKE orders;

-- 按范围分表
CREATE TABLE users_0 (id INT, ...);  -- id % 10 = 0
CREATE TABLE users_1 (id INT, ...);  -- id % 10 = 1
```

### 4.3. 范式与反范式

#### 4.3.1. 适度冗余
```sql
-- ❌ 过度范式化（需要 JOIN）
SELECT o.*, u.name, u.phone
FROM orders o
JOIN users u ON o.user_id = u.id;

-- ✅ 适度冗余（避免 JOIN）
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    user_name VARCHAR(50),  -- 冗余
    user_phone VARCHAR(20), -- 冗余
    ...
);
```

## 5. 配置优化

### 5.1. 连接池配置
```ini
[mysqld]
# 最大连接数
max_connections = 1000

# 连接超时
wait_timeout = 28800
interactive_timeout = 28800

# 连接队列
back_log = 500
```

### 5.2. 缓冲池配置
```ini
[mysqld]
# InnoDB 缓冲池（物理内存的 70-80%）
innodb_buffer_pool_size = 8G

# 缓冲池实例数（提高并发）
innodb_buffer_pool_instances = 8

# 查询缓存（MySQL 8.0 已移除）
query_cache_size = 0
query_cache_type = 0
```

### 5.3. 日志配置
```ini
[mysqld]
# 慢查询日志
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# 记录未使用索引的查询
log_queries_not_using_indexes = 1

# Binlog 配置
binlog_format = ROW
sync_binlog = 1
```

### 5.4. InnoDB 配置
```ini
[mysqld]
# 日志文件大小
innodb_log_file_size = 512M

# 日志缓冲区
innodb_log_buffer_size = 16M

# 刷新策略（1 最安全，2 性能最好）
innodb_flush_log_at_trx_commit = 1

# 每个表独立表空间
innodb_file_per_table = 1

# IO 线程
innodb_read_io_threads = 8
innodb_write_io_threads = 8
```

## 6. 监控与诊断

### 6.1. 慢查询分析
```bash
# 使用 mysqldumpslow 分析
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 使用 pt-query-digest
pt-query-digest /var/log/mysql/slow.log > slow_report.txt
```

### 6.2. 性能监控
```sql
-- 查看当前连接
SHOW PROCESSLIST;

-- 查看锁等待
SELECT * FROM information_schema.innodb_locks;
SELECT * FROM information_schema.innodb_lock_waits;

-- 查看事务
SELECT * FROM information_schema.innodb_trx;

-- 查看缓冲池状态
SHOW ENGINE INNODB STATUS;

-- 查看表状态
SHOW TABLE STATUS LIKE 'orders';
```

### 6.3. 关键指标
```sql
-- QPS/TPS
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Com_commit';

-- 缓存命中率
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';

-- 慢查询数量
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- 连接数
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
```

## 7. 实战案例

### 7.1. 案例 1：订单查询优化
```sql
-- ❌ 优化前（3.5 秒）
SELECT * FROM orders 
WHERE user_id = 1000 
  AND status IN (1, 2, 3)
  AND create_time >= '2024-01-01'
ORDER BY create_time DESC
LIMIT 20;

-- 分析问题
EXPLAIN ...
-- type: ALL, rows: 5000000

-- ✅ 优化方案
-- 1. 创建联合索引
CREATE INDEX idx_user_status_time ON orders(user_id, status, create_time);

-- 2. 优化查询（覆盖索引）
SELECT id, order_no, amount, create_time 
FROM orders 
WHERE user_id = 1000 
  AND status IN (1, 2, 3)
  AND create_time >= '2024-01-01'
ORDER BY create_time DESC
LIMIT 20;

-- 优化后（0.05 秒）
-- type: range, rows: 100
```

### 7.2. 案例 2：统计查询优化
```sql
-- ❌ 优化前（10 秒）
SELECT 
    DATE(create_time) AS date,
    COUNT(*) AS count,
    SUM(amount) AS total
FROM orders
WHERE create_time >= '2024-01-01'
GROUP BY DATE(create_time);

-- ✅ 优化方案
-- 1. 避免函数操作
SELECT 
    DATE_FORMAT(create_time, '%Y-%m-%d') AS date,
    COUNT(*) AS count,
    SUM(amount) AS total
FROM orders
WHERE create_time >= '2024-01-01'
GROUP BY date;

-- 2. 使用汇总表
CREATE TABLE order_daily_stats (
    date DATE PRIMARY KEY,
    count INT,
    total DECIMAL(15,2),
    updated_at TIMESTAMP
);

-- 定时任务更新汇总表
INSERT INTO order_daily_stats
SELECT 
    DATE(create_time),
    COUNT(*),
    SUM(amount),
    NOW()
FROM orders
WHERE DATE(create_time) = CURDATE()
ON DUPLICATE KEY UPDATE
    count = VALUES(count),
    total = VALUES(total),
    updated_at = NOW();

-- 查询汇总表（0.01 秒）
SELECT * FROM order_daily_stats 
WHERE date >= '2024-01-01';
```

### 7.3. 案例 3：大表 JOIN 优化
```sql
-- ❌ 优化前（30 秒）
SELECT 
    o.order_no,
    u.name,
    p.product_name,
    o.amount
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id
WHERE o.create_time >= '2024-01-01';

-- ✅ 优化方案
-- 1. 添加索引
CREATE INDEX idx_create_time ON orders(create_time);
CREATE INDEX idx_user_id ON orders(user_id);
CREATE INDEX idx_product_id ON orders(product_id);

-- 2. 使用覆盖索引 + 延迟关联
SELECT 
    o.order_no,
    o.user_name,    -- 冗余字段
    o.product_name, -- 冗余字段
    o.amount
FROM orders o
WHERE o.create_time >= '2024-01-01';

-- 优化后（0.5 秒）
```

## 8. 优化检查清单

### 8.1. 查询优化
- [ ] 使用 EXPLAIN 分析执行计划
- [ ] 避免 SELECT *，只查询需要的列
- [ ] 合理使用索引
- [ ] 避免索引失效场景
- [ ] 优化 JOIN 和子查询
- [ ] 分页查询使用延迟关联

### 8.2. 索引优化
- [ ] 为 WHERE、ORDER BY、GROUP BY 列建索引
- [ ] 使用联合索引遵循最左前缀
- [ ] 定期检查未使用的索引
- [ ] 索引列选择性 > 0.1
- [ ] 避免过多索引（影响写入性能）

### 8.3. 表结构优化
- [ ] 选择合适的字段类型
- [ ] 字段长度不要过大
- [ ] 适度使用冗余字段
- [ ] 大表考虑分区或分表
- [ ] 大字段独立存储

### 8.4. 配置优化
- [ ] innodb_buffer_pool_size 设置合理
- [ ] 开启慢查询日志
- [ ] 连接数配置合理
- [ ] 定期分析慢查询

## 9. 相关概念
- [[03-方法卡/数据库/MySQL大表数据删除方案\|MySQL大表数据删除方案]] - 数据清理
- [[03-方法卡/数据库/MySQL备份恢复方案\|MySQL备份恢复方案]] - 数据安全
- [[01-概念卡/数据库/InnoDB存储引擎原理\|InnoDB存储引擎原理]] - 底层机制
- [[01-概念卡/数据库/MySQL索引原理\|MySQL索引原理]] - 索引详解

## 10. 参考资源
- [MySQL 官方优化文档](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [高性能 MySQL](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)

