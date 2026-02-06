---
{"dg-publish":true,"permalink":"/03-方法卡/数据库/MySQL备份恢复方案/","tags":["数据备份","灾难恢复","数据安全","方法卡","tech/mysql"]}
---

## 1. 核心概念

### 1.1. 备份类型
- **物理备份**: 直接复制数据文件（快速，适合大数据量）
- **逻辑备份**: 导出 SQL 语句（灵活，跨平台）
- **全量备份**: 备份所有数据
- **增量备份**: 仅备份变化的数据
- **差异备份**: 备份自上次全量备份后的变化

### 1.2. 备份策略
- **RTO** (Recovery Time Objective): 恢复时间目标
- **RPO** (Recovery Point Objective): 恢复点目标（可接受的数据丢失时间）

## 2. 备份方案

### 2.1. 方案一：mysqldump（逻辑备份）

#### 2.1.1. 全库备份
```bash
# 备份所有数据库
mysqldump -u root -p \
  --all-databases \
  --single-transaction \
  --master-data=2 \
  --flush-logs \
  --routines \
  --triggers \
  --events \
  > /backup/all_databases_$(date +%Y%m%d_%H%M%S).sql

# 压缩备份
mysqldump -u root -p --all-databases | gzip > backup.sql.gz
```

**参数说明**:
- `--single-transaction`: InnoDB 一致性备份，不锁表
- `--master-data=2`: 记录 binlog 位置（用于主从复制）
- `--flush-logs`: 刷新日志，生成新的 binlog
- `--routines`: 备份存储过程和函数
- `--triggers`: 备份触发器
- `--events`: 备份事件调度器

#### 2.1.2. 单库备份
```bash
# 备份指定数据库
mysqldump -u root -p \
  --single-transaction \
  --databases mydb \
  > /backup/mydb_$(date +%Y%m%d).sql

# 备份指定表
mysqldump -u root -p mydb orders order_items \
  > /backup/mydb_orders_$(date +%Y%m%d).sql
```

#### 2.1.3. 备份脚本
```bash
#!/bin/bash
# mysql_backup.sh

BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
MYSQL_USER="backup_user"
MYSQL_PASS="backup_password"
RETENTION_DAYS=7

# 创建备份目录
mkdir -p ${BACKUP_DIR}

# 全量备份
mysqldump -u${MYSQL_USER} -p${MYSQL_PASS} \
  --all-databases \
  --single-transaction \
  --master-data=2 \
  --flush-logs \
  --routines \
  --triggers \
  --events \
  | gzip > ${BACKUP_DIR}/full_backup_${DATE}.sql.gz

# 检查备份是否成功
if [ $? -eq 0 ]; then
    echo "Backup successful: full_backup_${DATE}.sql.gz"
    
    # 删除过期备份
    find ${BACKUP_DIR} -name "full_backup_*.sql.gz" -mtime +${RETENTION_DAYS} -delete
else
    echo "Backup failed!" >&2
    exit 1
fi
```

#### 2.1.4. 恢复数据
```bash
# 恢复全部数据库
mysql -u root -p < backup.sql

# 恢复压缩备份
gunzip < backup.sql.gz | mysql -u root -p

# 恢复指定数据库
mysql -u root -p mydb < mydb_backup.sql

# 恢复时跳过错误
mysql -u root -p --force < backup.sql
```

### 2.2. 方案二：XtraBackup（物理备份）

#### 2.2.1. 安装 XtraBackup
```bash
# CentOS/RHEL
yum install percona-xtrabackup-80

# Ubuntu/Debian
apt-get install percona-xtrabackup-80
```

#### 2.2.2. 全量备份
```bash
# 全量备份
xtrabackup --backup \
  --user=root \
  --password=password \
  --target-dir=/backup/full_$(date +%Y%m%d)

# 压缩备份
xtrabackup --backup \
  --user=root \
  --password=password \
  --compress \
  --compress-threads=4 \
  --target-dir=/backup/full_compressed
```

#### 2.2.3. 增量备份
```bash
# 第一次全量备份
xtrabackup --backup \
  --target-dir=/backup/base

# 第一次增量备份
xtrabackup --backup \
  --target-dir=/backup/inc1 \
  --incremental-basedir=/backup/base

# 第二次增量备份
xtrabackup --backup \
  --target-dir=/backup/inc2 \
  --incremental-basedir=/backup/inc1
```

#### 2.2.4. 恢复数据
```bash
# 准备全量备份
xtrabackup --prepare \
  --target-dir=/backup/full

# 准备增量备份
xtrabackup --prepare \
  --apply-log-only \
  --target-dir=/backup/base

xtrabackup --prepare \
  --apply-log-only \
  --target-dir=/backup/base \
  --incremental-dir=/backup/inc1

xtrabackup --prepare \
  --target-dir=/backup/base \
  --incremental-dir=/backup/inc2

# 停止 MySQL
systemctl stop mysql

# 恢复数据
xtrabackup --copy-back \
  --target-dir=/backup/base

# 修改权限
chown -R mysql:mysql /var/lib/mysql

# 启动 MySQL
systemctl start mysql
```

### 2.3. 方案三：Binlog 增量备份

#### 2.3.1. 启用 Binlog
```ini
# my.cnf
[mysqld]
server-id = 1
log-bin = /var/log/mysql/mysql-bin
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 100M
```

#### 2.3.2. 备份 Binlog
```bash
# 刷新 binlog（生成新文件）
mysql -u root -p -e "FLUSH LOGS"

# 复制 binlog 文件
cp /var/log/mysql/mysql-bin.000001 /backup/binlog/

# 使用 mysqlbinlog 导出
mysqlbinlog /var/log/mysql/mysql-bin.000001 > /backup/binlog.sql
```

#### 2.3.3. 基于时间点恢复
```bash
# 恢复到指定时间点
mysqlbinlog --stop-datetime="2024-01-15 10:30:00" \
  /var/log/mysql/mysql-bin.000001 | mysql -u root -p

# 恢复指定时间段
mysqlbinlog \
  --start-datetime="2024-01-15 09:00:00" \
  --stop-datetime="2024-01-15 10:00:00" \
  /var/log/mysql/mysql-bin.* | mysql -u root -p
```

#### 2.3.4. 基于位置恢复
```bash
# 查看 binlog 位置
mysqlbinlog /var/log/mysql/mysql-bin.000001 | grep "at"

# 恢复到指定位置
mysqlbinlog --stop-position=12345 \
  /var/log/mysql/mysql-bin.000001 | mysql -u root -p

# 跳过错误事务（如误删除）
mysqlbinlog \
  --start-position=12345 \
  --stop-position=12400 \
  /var/log/mysql/mysql-bin.000001 > skip.sql
# 手动编辑 skip.sql，删除错误语句后执行
```

### 2.4. 方案四：主从复制备份

#### 2.4.1. 配置主从复制
```sql
-- 主库配置
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

-- 查看主库状态
SHOW MASTER STATUS;

-- 从库配置
CHANGE MASTER TO
  MASTER_HOST='192.168.1.100',
  MASTER_USER='repl',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

START SLAVE;
SHOW SLAVE STATUS\G
```

#### 2.4.2. 从库备份
```bash
# 在从库执行备份（不影响主库）
mysqldump -u root -p \
  --single-transaction \
  --all-databases \
  > /backup/slave_backup.sql
```

## 3. 备份策略设计

### 3.1. 策略一：全量 + 增量（推荐）
```bash
# 每周日全量备份
0 2 * * 0 /scripts/full_backup.sh

# 每天增量备份（binlog）
0 3 * * 1-6 /scripts/incremental_backup.sh

# 每小时备份 binlog
0 * * * * /scripts/binlog_backup.sh
```

**特点**:
- RPO: 1 小时
- RTO: 2-4 小时
- 存储成本: 中等

### 3.2. 策略二：全量 + 差异
```bash
# 每周日全量备份
0 2 * * 0 xtrabackup --backup --target-dir=/backup/full_$(date +%u)

# 每天差异备份
0 3 * * 1-6 xtrabackup --backup --incremental-basedir=/backup/full_0 \
  --target-dir=/backup/diff_$(date +%u)
```

**特点**:
- RPO: 1 天
- RTO: 1-2 小时
- 存储成本: 较高

### 3.3. 策略三：实时备份（高可用）
```bash
# 主从复制 + 延迟从库
-- 延迟 1 小时的从库（防止误操作）
CHANGE MASTER TO MASTER_DELAY = 3600;

# 定期快照
0 */6 * * * /scripts/snapshot_backup.sh
```

**特点**:
- RPO: 接近 0
- RTO: < 1 小时
- 成本: 高

## 4. 完整备份恢复流程

### 4.1. 场景一：数据库崩溃恢复
```bash
# 1. 停止 MySQL
systemctl stop mysql

# 2. 恢复最近的全量备份
xtrabackup --prepare --target-dir=/backup/full_20240115
xtrabackup --copy-back --target-dir=/backup/full_20240115

# 3. 应用增量备份
mysqlbinlog /backup/binlog/mysql-bin.* | mysql -u root -p

# 4. 修改权限并启动
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql

# 5. 验证数据
mysql -u root -p -e "SELECT COUNT(*) FROM mydb.orders"
```

### 4.2. 场景二：误删除数据恢复
```bash
# 1. 找到误删除的时间点
# 假设在 2024-01-15 14:30:00 误删除

# 2. 恢复到误删除前
mysqlbinlog --stop-datetime="2024-01-15 14:29:59" \
  /var/log/mysql/mysql-bin.000010 > before_delete.sql

# 3. 跳过删除操作，恢复后续数据
mysqlbinlog --start-datetime="2024-01-15 14:31:00" \
  /var/log/mysql/mysql-bin.000010 > after_delete.sql

# 4. 应用恢复
mysql -u root -p < before_delete.sql
mysql -u root -p < after_delete.sql
```

### 4.3. 场景三：单表恢复
```bash
# 1. 从备份中提取单表
mysql -u root -p -e "CREATE DATABASE temp_restore"
mysql -u root -p temp_restore < full_backup.sql

# 2. 导出目标表
mysqldump -u root -p temp_restore orders > orders_restore.sql

# 3. 恢复到生产库
mysql -u root -p production < orders_restore.sql

# 4. 清理临时库
mysql -u root -p -e "DROP DATABASE temp_restore"
```

## 5. 监控与验证

### 5.1. 备份验证
```bash
#!/bin/bash
# 验证备份完整性

BACKUP_FILE="/backup/full_backup.sql.gz"

# 1. 检查文件是否存在
if [ ! -f "$BACKUP_FILE" ]; then
    echo "Backup file not found!"
    exit 1
fi

# 2. 检查文件大小
SIZE=$(stat -f%z "$BACKUP_FILE" 2>/dev/null || stat -c%s "$BACKUP_FILE")
if [ $SIZE -lt 1000000 ]; then
    echo "Backup file too small: $SIZE bytes"
    exit 1
fi

# 3. 测试解压
gunzip -t "$BACKUP_FILE"
if [ $? -ne 0 ]; then
    echo "Backup file corrupted!"
    exit 1
fi

# 4. 测试恢复（在测试环境）
gunzip < "$BACKUP_FILE" | mysql -u root -p test_db
if [ $? -eq 0 ]; then
    echo "Backup validation successful"
else
    echo "Backup validation failed!"
    exit 1
fi
```

### 5.2. 监控指标
```sql
-- 检查备份状态
SELECT 
    table_schema,
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
GROUP BY table_schema;

-- 检查 binlog 大小
SHOW BINARY LOGS;

-- 检查主从延迟
SHOW SLAVE STATUS\G
```

## 6. 最佳实践

### 6.1. ✅ 推荐做法
1. **3-2-1 原则**: 3 份副本，2 种介质，1 份异地
2. **定期测试**: 每月至少测试一次恢复流程
3. **自动化**: 使用脚本自动化备份和验证
4. **监控告警**: 备份失败立即告警
5. **文档记录**: 记录备份策略和恢复步骤
6. **权限控制**: 备份文件加密，限制访问权限

### 6.2. ❌ 避免做法
1. 仅依赖单一备份方式
2. 不测试恢复流程
3. 备份文件与数据库在同一磁盘
4. 不验证备份完整性
5. 没有异地备份

## 7. 备份工具对比

| 工具 | 类型 | 速度 | 锁表 | 增量 | 适用场景 |
|------|------|------|------|------|----------|
| mysqldump | 逻辑 | 慢 | 部分 | 否 | 小型数据库 |
| XtraBackup | 物理 | 快 | 否 | 是 | 大型数据库 |
| mysqlpump | 逻辑 | 中 | 否 | 否 | 并行备份 |
| MySQL Enterprise Backup | 物理 | 快 | 否 | 是 | 企业级 |

## 8. 成本估算

### 8.1. 存储成本
```
全量备份: 100GB
增量备份: 10GB/天
保留周期: 30 天

总存储 = 100GB × 4 (每周全量) + 10GB × 30 = 700GB
```

### 8.2. 时间成本
- mysqldump: ~1GB/分钟
- XtraBackup: ~5GB/分钟
- 恢复时间: 备份时间的 2-3 倍

## 9. 相关概念
- [[03-方法卡/数据库/MySQL大表数据删除方案\|MySQL大表数据删除方案]] - 数据清理
- [[01-概念卡/数据库/MySQL主从复制架构\|MySQL主从复制架构]] - 高可用
- [[03-方法卡/数据库/MySQL灾难恢复演练\|MySQL灾难恢复演练]] - 容灾方案
- [[03-方法卡/数据库/MySQL性能优化\|MySQL性能优化]] - 性能调优

## 10. 参考资源
- [MySQL 官方备份文档](https://dev.mysql.com/doc/refman/8.0/en/backup-and-recovery.html)
- [Percona XtraBackup 文档](https://docs.percona.com/percona-xtrabackup/)

