---
{"dg-publish":true,"permalink":"/03-方法卡/数据库/MySQL大表数据删除方案/","tags":["数据库优化","大表处理","数据清理","方法卡","tech/mysql"]}
---

## 1. 问题背景

### 1.1. InnoDB 存储特性
InnoDB 存储引擎由逻辑存储结构和物理存储结构组成，数据存储的最小单位是 **Page（页）**。

### 1.2. 删除数据的真相
当执行 DELETE 操作时，MySQL **不会立即回收存储空间**，而是将这部分空间标记为"可复用"状态。删除的数据在物理上仍然存在，只是被标记为可覆盖。

## 2. 核心问题

### 2.1. 空间不释放
```sql
-- 删除 100 万条数据
DELETE FROM orders WHERE create_time < '2023-01-01';

-- 查看表大小，发现空间未减少
SELECT 
    table_name,
    ROUND(data_length/1024/1024, 2) AS data_mb,
    ROUND(data_free/1024/1024, 2) AS free_mb
FROM information_schema.tables
WHERE table_name = 'orders';
```

**原因**: 删除的空间被标记为"可用"，但未归还给操作系统。

### 2.2. 表碎片问题
- **内部碎片**: 页内部的空闲空间
- **外部碎片**: 页之间的不连续空间
- **影响**: 降低查询性能，增加 I/O 开销

### 2.3. 性能开销
- 大量删除操作产生 Undo Log
- 碎片整理需要锁表，影响业务
- 扫描包含大量"已删除"标记的数据页

## 3. 解决方案

### 3.1. 方案一：分区表 + 归档 + 定期清理（推荐）

#### 3.1.1. 分区表设计 (不推荐)
```sql
-- 按时间分区（每月一个分区）
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(64),
    create_time DATETIME,
    -- 其他字段
    INDEX idx_create_time (create_time)
) PARTITION BY RANGE (TO_DAYS(create_time)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    PARTITION p202403 VALUES LESS THAN (TO_DAYS('2024-04-01')),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

**优势**:
- 避免扫描无效的过期数据
- 删除整个分区速度快
- 便于数据归档管理


#### 3.1.2. 数据归档
```sql
-- 创建归档表（结构相同）
CREATE TABLE orders_archive LIKE orders;

-- 归档 3 个月前的数据
INSERT INTO orders_archive 
SELECT * FROM orders 
WHERE create_time < DATE_SUB(NOW(), INTERVAL 3 MONTH);

-- 删除分区（快速释放空间）
ALTER TABLE orders DROP PARTITION p202401;
```

**优势**:
- 保留历史数据，支持数据恢复
- 主表保持精简，查询性能高
- 归档表可迁移到冷存储

#### 3.1.3. 定期清理
```sql
-- 批量删除（控制每次删除量）
DELIMITER $
CREATE PROCEDURE batch_delete_orders()
BEGIN
    DECLARE affected_rows INT DEFAULT 1;
    
    WHILE affected_rows > 0 DO
        DELETE FROM orders 
        WHERE create_time < DATE_SUB(NOW(), INTERVAL 6 MONTH)
        LIMIT 1000;  -- 每次删除 1000 条
        
        SET affected_rows = ROW_COUNT();
        
        -- 暂停 100ms，避免持续占用资源
        SELECT SLEEP(0.1);
    END WHILE;
END$
DELIMITER ;

-- 执行清理
CALL batch_delete_orders();
```

**要点**:
- 控制每批删除数量（建议 500-5000 条）
- 批次间增加延迟，降低系统压力
- 在业务低峰期执行

### 3.2. 方案二：碎片整理

#### 3.2.1. OPTIMIZE TABLE（推荐）
```sql
-- 重建表，回收碎片空间
OPTIMIZE TABLE orders;
```

**特点**:
- 重建表结构，彻底回收空间
- 会锁表，影响业务（需在低峰期执行）
- 适用于 InnoDB 和 MyISAM

#### 3.2.2. ALTER TABLE（在线 DDL）
```sql
-- MySQL 5.6+ 支持在线 DDL
ALTER TABLE orders ENGINE=InnoDB;
```

**特点**:
- 支持在线操作，不完全锁表
- 需要额外的磁盘空间（临时表）
- 适合大表操作

#### 3.2.3. 查看碎片情况
```sql
-- 检查表碎片
SELECT 
    table_name,
    ROUND(data_length/1024/1024, 2) AS data_mb,
    ROUND(index_length/1024/1024, 2) AS index_mb,
    ROUND(data_free/1024/1024, 2) AS free_mb,
    ROUND(data_free/(data_length + index_length) * 100, 2) AS frag_pct
FROM information_schema.tables
WHERE table_schema = 'your_database'
    AND data_free > 0
ORDER BY frag_pct DESC;
```

### 3.3. 方案三：逻辑删除（软删除）

```sql
-- 添加删除标记字段
ALTER TABLE orders ADD COLUMN is_deleted TINYINT DEFAULT 0;
ALTER TABLE orders ADD INDEX idx_is_deleted (is_deleted);

-- 逻辑删除
UPDATE orders SET is_deleted = 1 
WHERE create_time < '2023-01-01';

-- 查询时过滤
SELECT * FROM orders WHERE is_deleted = 0;
```

**优势**:
- 数据可恢复
- 无物理删除开销
- 支持审计和追溯

**劣势**:
- 表空间持续增长
- 查询需要额外过滤条件
- 索引包含无效数据

## 4. 最佳实践

### 4.1. 设计阶段
```sql
-- 建表时考虑分区策略
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    create_time DATETIME NOT NULL,
    is_deleted TINYINT DEFAULT 0,
    INDEX idx_create_time (create_time),
    INDEX idx_is_deleted (is_deleted)
) PARTITION BY RANGE (TO_DAYS(create_time)) (
    -- 分区定义
);
```

### 4.2. 删除策略
- **小批量**: 每次删除 500-5000 条
- **低峰期**: 凌晨或业务低谷时段
- **加延迟**: 批次间暂停 50-200ms
- **监控**: 观察主从延迟和系统负载

### 4.3. 归档策略
```bash
#!/bin/bash
# 自动归档脚本

# 归档 3 个月前的数据
mysql -e "
    INSERT INTO orders_archive 
    SELECT * FROM orders 
    WHERE create_time < DATE_SUB(NOW(), INTERVAL 3 MONTH)
    AND id NOT IN (SELECT id FROM orders_archive);
"

# 删除已归档的数据
mysql -e "
    DELETE FROM orders 
    WHERE create_time < DATE_SUB(NOW(), INTERVAL 3 MONTH)
    LIMIT 10000;
"
```

### 4.4. 碎片整理计划
```sql
-- 定期检查碎片率
-- 碎片率 > 30% 时执行整理

-- 创建维护任务（每周日凌晨 3 点）
CREATE EVENT optimize_tables
ON SCHEDULE EVERY 1 WEEK
STARTS '2024-01-07 03:00:00'
DO
BEGIN
    OPTIMIZE TABLE orders;
    OPTIMIZE TABLE order_items;
END;
```

## 5. 性能对比

### 5.1. 删除方式对比
| 方式 | 速度 | 空间回收 | 业务影响 | 适用场景 |
|------|------|----------|----------|----------|
| DELETE | 慢 | 否 | 中 | 少量数据 |
| TRUNCATE | 快 | 是 | 高（锁表） | 清空整表 |
| DROP PARTITION | 极快 | 是 | 低 | 分区表 |
| 逻辑删除 | 快 | 否 | 低 | 需要恢复 |

### 5.2. 碎片整理对比
| 方式 | 速度 | 锁表 | 空间需求 | 推荐度 |
|------|------|------|----------|--------|
| OPTIMIZE TABLE | 中 | 是 | 1x | ⭐⭐⭐ |
| ALTER TABLE | 慢 | 部分 | 2x | ⭐⭐ |
| 导出导入 | 慢 | 是 | 2x | ⭐ |

## 6. 监控指标

### 6.1. 关键指标
```sql
-- 表大小监控
SELECT 
    table_name,
    ROUND((data_length + index_length)/1024/1024, 2) AS total_mb,
    table_rows,
    ROUND(data_free/1024/1024, 2) AS free_mb
FROM information_schema.tables
WHERE table_schema = DATABASE();

-- 碎片率监控
SELECT 
    table_name,
    ROUND(data_free/(data_length + index_length) * 100, 2) AS frag_pct
FROM information_schema.tables
WHERE table_schema = DATABASE()
    AND data_free > 0
ORDER BY frag_pct DESC;
```

### 6.2. 告警阈值
- 碎片率 > 30%: 建议整理
- 表大小增长 > 50%/月: 检查删除策略
- 主从延迟 > 5s: 降低删除频率

## 7. 注意事项

### 7.1. ⚠️ 风险提示
1. **OPTIMIZE TABLE 会锁表**: 必须在业务低峰期执行
2. **需要足够磁盘空间**: 至少预留表大小的 2 倍空间
3. **主从延迟**: 大量删除会导致主从延迟增加
4. **备份**: 执行大规模删除前务必备份

### 7.2. ✅ 操作检查清单
- [ ] 确认业务低峰期时间窗口
- [ ] 检查磁盘剩余空间（> 2x 表大小）
- [ ] 备份相关数据
- [ ] 评估主从延迟影响
- [ ] 准备回滚方案
- [ ] 监控系统负载

## 8. 相关概念
- [[01-概念卡/数据库/InnoDB存储引擎原理\|InnoDB存储引擎原理]] - 存储机制
- [[03-方法卡/数据库/MySQL性能优化\|MySQL性能优化]] - 性能调优
- [[03-方法卡/数据库/MySQL备份恢复方案\|MySQL备份恢复方案]] - 数据安全

## 9. 参考资源
- [MySQL 官方文档 - OPTIMIZE TABLE](https://dev.mysql.com/doc/refman/8.0/en/optimize-table.html)
- [MySQL 分区表最佳实践](https://dev.mysql.com/doc/refman/8.0/en/partitioning.html)
