---
{"dg-publish":true,"permalink":"/66.归档发布/06.数据库/MySQL开发规范清单/"}
---

#数据库 #mysql #最佳实践

## 1. 命名规范

- 库名、表名、字段名：小写 + 下划线，不超过 32 字符，见名知意
- 禁止拼音英文混用，禁止使用 MySQL 保留字

索引命名：
```
普通索引：idx_表名_字段名   → idx_user_name
唯一索引：uk_表名_字段名    → uk_user_email
临时表：  tmp_表名_20240121
备份表：  bak_表名_20240121
```

## 2. 表设计

```sql
CREATE TABLE user (
    id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(50)     NOT NULL DEFAULT '' COMMENT '用户名',
    status     TINYINT         NOT NULL DEFAULT 0  COMMENT '状态：0-正常 1-禁用',
    created_at TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

- 主键用 `BIGINT UNSIGNED` 自增，禁止用字符串、UUID、MD5
- 单表列数 < 50，数据量 < 500 万行
- 禁止外键、禁止分区表、禁止预留字段

## 3. 字段设计

字段必须 NOT NULL 并给默认值，NULL 字段索引需要额外空间，复合索引里有 NULL 会失效。

```sql
-- 金额用分存整数，不用 FLOAT/DOUBLE
amount BIGINT NOT NULL DEFAULT 0 COMMENT '金额（分）'

-- 精确小数用 DECIMAL
price DECIMAL(10,2) NOT NULL DEFAULT 0.00

-- IP 地址存整数，用 INET_ATON()/INET_NTOA() 转换
ip INT UNSIGNED NOT NULL DEFAULT 0

-- 状态类字段用 TINYINT，禁止用 ENUM
status TINYINT NOT NULL DEFAULT 0
```

禁止使用 TEXT、BLOB（万级以下小表可考虑）。

## 4. 索引规范

- 单表索引 ≤ 5 个，复合索引字段数 ≤ 5 个
- 区分度高的字段放复合索引前面
- 更新频繁、区分度低的字段不建索引

索引失效的常见场景：

```sql
-- 索引列用了函数
WHERE YEAR(created_at) = 2024        -- ❌
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'  -- ✅

-- 前置通配符
WHERE name LIKE '%zhang%'  -- ❌
WHERE name LIKE 'zhang%'   -- ✅

-- 隐式类型转换（phone 是 VARCHAR）
WHERE phone = 13800138000    -- ❌
WHERE phone = '13800138000'  -- ✅

-- OR 中有一列无索引，整体失效
WHERE name = 'zhang' OR age = 20  -- age 无索引则 ❌
```

## 5. SQL 规范

```sql
-- 禁止 SELECT *，显式指定列
SELECT id, name, age FROM user;

-- 批量插入，显式指定列
INSERT INTO user (name, age) VALUES ('zhang', 20), ('li', 25);

-- UPDATE/DELETE 的 WHERE 必须走索引，否则锁表
UPDATE user SET status = 1 WHERE id = 1;  -- ✅ 主键
UPDATE user SET status = 1 WHERE name = 'zhang';  -- ❌ name 无索引

-- 逻辑删除代替物理删除
UPDATE user SET deleted = 1 WHERE id = 1;

-- 用 IN 代替 OR
WHERE id IN (1, 2, 3)  -- ✅
WHERE id = 1 OR id = 2 OR id = 3  -- ❌

-- 深分页优化
-- ❌ 慢
SELECT * FROM user LIMIT 100000, 10;
-- ✅ 延迟关联
SELECT u.* FROM user u
INNER JOIN (SELECT id FROM user WHERE status = 1 LIMIT 100000, 10) t ON u.id = t.id;
-- ✅ 记录上次位置（更推荐）
SELECT * FROM user WHERE id > 100000 LIMIT 10;

-- COUNT 用 COUNT(*)
SELECT COUNT(*) FROM user;
```

## 6. 事务规范

事务要短，不要在事务里调外部接口、做复杂计算、处理大量数据。

```sql
-- ✅ 简单快速
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

JOIN 不超过 3 个表，JOIN 字段必须有索引且类型一致。子查询尽量改写成 JOIN。

## 7. 大表变更

表结构变更用 `pt-online-schema-change`，避免锁表：

```bash
pt-online-schema-change \
  --alter "ADD COLUMN email VARCHAR(50) NOT NULL DEFAULT ''" \
  D=mydb,t=user \
  --execute
```

核心业务变更在凌晨执行，必须准备回滚方案。

## 8. 安全
- 密码加密存储，敏感字段加密或脱敏
- 必须用预编译语句，禁止拼接 SQL
```java
// ✅
PreparedStatement ps = conn.prepareStatement("SELECT * FROM user WHERE name = ?");
ps.setString(1, userName);

// ❌ SQL 注入风险
String sql = "SELECT * FROM user WHERE name = '" + userName + "'";
```
## 相关链接
- [[66.归档发布/06.数据库/JDBC连接参数优化方法\|JDBC连接参数优化方法]]
