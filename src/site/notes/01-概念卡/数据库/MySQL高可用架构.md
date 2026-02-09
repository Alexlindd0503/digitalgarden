---
{"dg-publish":true,"permalink":"/01-概念卡/数据库/MySQL高可用架构/","tags":["高可用","架构设计","容灾","概念卡","tech/mysql"]}
---

## 1. 核心概念
高可用（High Availability）是指系统能够持续运行，在故障发生时快速恢复，保证业务连续性。目标是实现 **99.99%**（年停机 < 53 分钟）或更高的可用性。

## 2. 可用性等级

| 等级 | 可用性 | 年停机时间 | 适用场景 |
|------|--------|------------|----------|
| 基础 | 99% | 3.65 天 | 测试环境 |
| 标准 | 99.9% | 8.76 小时 | 一般业务 |
| 高可用 | 99.99% | 52.56 分钟 | 重要业务 |
| 极高可用 | 99.999% | 5.26 分钟 | 核心业务 |

## 3. 高可用架构方案

### 3.1. 主从复制 + 手动切换

#### 3.1.1. 架构图
```
Master ──→ Slave1
       └──→ Slave2
```

#### 3.1.2. 特点
- **可用性**: 99.9%
- **RTO**: 5-30 分钟（手动切换）
- **RPO**: 0-5 分钟
- **成本**: 低

#### 3.1.3. 故障切换
```bash
# 手动提升从库
mysql -h slave1 -e "STOP SLAVE; RESET MASTER; SET GLOBAL read_only=0;"

# 应用切换连接
# 修改配置文件或 DNS
```

**优点**: 简单、成本低
**缺点**: 需要人工介入、RTO 较长

### 3.2. MHA（Master High Availability）

#### 3.2.1. 架构图
```
     MHA Manager
         |
    监控和切换
         |
Master ──┬──→ Slave1
         └──→ Slave2
```

#### 3.2.2. 特点
- **可用性**: 99.95%
- **RTO**: 10-30 秒
- **RPO**: 0（半同步）
- **成本**: 中

#### 3.2.3. 配置示例
```ini
# /etc/mha/app1.cnf
[server default]
manager_workdir=/var/log/mha/app1
manager_log=/var/log/mha/app1/manager.log
ssh_user=root
repl_user=repl
repl_password=password

[server1]
hostname=192.168.1.100
candidate_master=1

[server2]
hostname=192.168.1.101
candidate_master=1

[server3]
hostname=192.168.1.102
no_master=1
```

**优点**: 自动切换、成熟稳定
**缺点**: 需要额外管理节点、不支持多主

### 3.3. MySQL Group Replication（MGR）

#### 3.3.1. 架构图
```
Node1 ←→ Node2 ←→ Node3
  ↑        ↑        ↑
  └────────┴────────┘
     Paxos 协议
```

#### 3.3.2. 特点
- **可用性**: 99.99%
- **RTO**: < 10 秒
- **RPO**: 0
- **成本**: 中高

#### 3.3.3. 配置示例
```ini
[mysqld]
# 基础配置
server_id = 1
gtid_mode = ON
enforce_gtid_consistency = ON

# MGR 配置
plugin_load_add = 'group_replication.so'
group_replication_group_name = "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
group_replication_start_on_boot = OFF
group_replication_local_address = "192.168.1.100:33061"
group_replication_group_seeds = "192.168.1.100:33061,192.168.1.101:33061,192.168.1.102:33061"
group_replication_bootstrap_group = OFF
```

**优点**: 自动故障切换、多主模式、数据强一致
**缺点**: 性能开销、网络要求高

### 3.4. Galera Cluster（PXC）

#### 3.4.1. 架构图
```
Node1 ←→ Node2 ←→ Node3
  ↑        ↑        ↑
  └────────┴────────┘
   多主同步复制
```

#### 3.4.2. 特点
- **可用性**: 99.99%
- **RTO**: < 5 秒
- **RPO**: 0
- **成本**: 高

#### 3.4.3. 配置示例
```ini
[mysqld]
# Galera 配置
wsrep_on = ON
wsrep_provider = /usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address = "gcomm://192.168.1.100,192.168.1.101,192.168.1.102"
wsrep_cluster_name = "my_cluster"
wsrep_node_address = "192.168.1.100"
wsrep_node_name = "node1"
wsrep_sst_method = xtrabackup-v2
```

**优点**: 真正的多主、数据强一致、自动节点管理
**缺点**: 性能开销大、不支持大事务

### 3.5. 云原生方案（RDS）

#### 3.5.1. 架构图
```
Primary ──→ Standby (同城)
   └────────→ Standby (异地)
```

#### 3.5.2. 特点
- **可用性**: 99.95% - 99.99%
- **RTO**: < 60 秒
- **RPO**: 0
- **成本**: 高

**优点**: 托管服务、自动运维、多可用区
**缺点**: 成本高、厂商锁定

## 4. 架构对比

| 方案 | 可用性 | RTO | RPO | 复杂度 | 成本 | 推荐场景 |
|------|--------|-----|-----|--------|------|----------|
| 主从 + 手动 | 99.9% | 5-30分钟 | 0-5分钟 | 低 | 低 | 小型业务 |
| MHA | 99.95% | 10-30秒 | 0 | 中 | 中 | 中型业务 |
| MGR | 99.99% | <10秒 | 0 | 中高 | 中高 | 大型业务 |
| Galera | 99.99% | <5秒 | 0 | 高 | 高 | 金融业务 |
| RDS | 99.99% | <60秒 | 0 | 低 | 高 | 云上业务 |

## 5. 高可用组件

### 5.1. 负载均衡

#### 5.1.1. ProxySQL
```sql
-- 配置读写分离
INSERT INTO mysql_servers(hostgroup_id, hostname, port) 
VALUES (0, '192.168.1.100', 3306);  -- 写组

INSERT INTO mysql_servers(hostgroup_id, hostname, port) 
VALUES (1, '192.168.1.101', 3306);  -- 读组
INSERT INTO mysql_servers(hostgroup_id, hostname, port) 
VALUES (1, '192.168.1.102', 3306);

-- 配置路由规则
INSERT INTO mysql_query_rules(rule_id, active, match_pattern, destination_hostgroup)
VALUES (1, 1, '^SELECT.*FOR UPDATE', 0);  -- 写
INSERT INTO mysql_query_rules(rule_id, active, match_pattern, destination_hostgroup)
VALUES (2, 1, '^SELECT', 1);  -- 读

LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
```

#### 5.1.2. HAProxy
```conf
# haproxy.cfg
frontend mysql_frontend
    bind *:3306
    mode tcp
    default_backend mysql_backend

backend mysql_backend
    mode tcp
    balance roundrobin
    option mysql-check user haproxy
    server mysql1 192.168.1.100:3306 check
    server mysql2 192.168.1.101:3306 check backup
```

### 5.2. 虚拟 IP（VIP）

#### 5.2.1. Keepalived
```conf
# keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    
    virtual_ipaddress {
        192.168.1.200
    }
    
    track_script {
        chk_mysql
    }
}

vrrp_script chk_mysql {
    script "/usr/local/bin/check_mysql.sh"
    interval 2
    weight -20
}
```

### 5.3. 监控告警

#### 5.3.1. 关键指标
```sql
-- 主从延迟
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master

-- 连接数
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';

-- QPS/TPS
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Com_commit';

-- 慢查询
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

#### 5.3.2. Prometheus 监控
```yaml
# mysqld_exporter
scrape_configs:
  - job_name: 'mysql'
    static_configs:
      - targets: ['192.168.1.100:9104']
        labels:
          instance: 'mysql-master'
      - targets: ['192.168.1.101:9104']
        labels:
          instance: 'mysql-slave1'
```

## 6. 故障切换流程

### 6.1. 自动切换（MHA）
```
1. MHA Manager 检测到主库故障
   ↓
2. 选择最新的从库作为新主库
   ↓
3. 应用差异 relay log
   ↓
4. 提升从库为主库
   ↓
5. 其他从库指向新主库
   ↓
6. 更新 VIP 或 DNS
   ↓
7. 发送告警通知
```

### 6.2. 手动切换
```bash
#!/bin/bash
# manual_failover.sh

OLD_MASTER="192.168.1.100"
NEW_MASTER="192.168.1.101"
VIP="192.168.1.200"

# 1. 停止旧主库（如果还在运行）
ssh root@$OLD_MASTER "systemctl stop mysql"

# 2. 提升新主库
mysql -h $NEW_MASTER <<EOF
STOP SLAVE;
RESET MASTER;
SET GLOBAL read_only = 0;
EOF

# 3. 切换 VIP
ssh root@$OLD_MASTER "ip addr del $VIP/24 dev eth0"
ssh root@$NEW_MASTER "ip addr add $VIP/24 dev eth0"

# 4. 验证
mysql -h $VIP -e "SELECT @@hostname"

echo "切换完成"
```

## 7. 数据一致性

### 7.1. 半同步复制
```sql
-- 主库配置
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- 1秒超时

-- 从库配置
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

### 7.2. 数据校验
```bash
# 使用 pt-table-checksum
pt-table-checksum \
  --host=192.168.1.100 \
  --databases=mydb \
  --replicate=percona.checksums

# 修复不一致
pt-table-sync \
  --execute \
  --sync-to-master \
  h=192.168.1.101
```

## 8. 容灾方案

### 8.1. 同城双活
```
机房 A                机房 B
Master ←─────────→ Master
  ↓                    ↓
Slave1              Slave2
```

**特点**: 
- 两个机房都可写
- 网络延迟低（< 5ms）
- 防止单机房故障

### 8.2. 异地多活
```
北京机房          上海机房          深圳机房
Master1 ←────→ Master2 ←────→ Master3
  ↓               ↓               ↓
Slave1          Slave2          Slave3
```

**特点**:
- 多地域部署
- 就近访问
- 防止区域性灾难

## 9. 最佳实践

### 9.1. ✅ 推荐做法
1. 使用半同步复制保证数据安全
2. 部署监控和告警系统
3. 定期演练故障切换
4. 使用 GTID 简化复制管理
5. 配置自动故障切换
6. 多可用区部署
7. 定期备份和验证

### 9.2. ❌ 避免做法
1. 单点部署
2. 不监控主从延迟
3. 不测试故障切换
4. 手动管理复制位点
5. 忽略数据一致性
6. 所有节点在同一机房
7. 不备份或不验证备份

## 10. 成本估算

### 10.1. 硬件成本
```
主从架构（3 节点）:
- 服务器: 3 × 2万 = 6万
- 存储: 3 × 1万 = 3万
- 网络: 1万
总计: 10万

MGR 架构（3 节点）:
- 服务器: 3 × 3万 = 9万
- 存储: 3 × 2万 = 6万
- 网络: 2万
总计: 17万
```

### 10.2. 运维成本
```
- DBA 人力: 30万/年
- 监控工具: 5万/年
- 培训演练: 3万/年
总计: 38万/年
```

## 11. 选型建议

### 11.1. 小型业务（< 1000 QPS）
- **方案**: 主从 + 手动切换
- **成本**: 低
- **可用性**: 99.9%

### 11.2. 中型业务（1000-10000 QPS）
- **方案**: MHA 或 MGR
- **成本**: 中
- **可用性**: 99.95%-99.99%

### 11.3. 大型业务（> 10000 QPS）
- **方案**: MGR 或 Galera
- **成本**: 高
- **可用性**: 99.99%

### 11.4. 云上业务
- **方案**: RDS 高可用版
- **成本**: 中高
- **可用性**: 99.95%-99.99%

## 12. 相关概念
- [[01-概念卡/数据库/MySQL主从复制架构\|MySQL主从复制架构]] - 复制原理
- [[03-方法卡/数据库/MySQL备份恢复方案\|MySQL备份恢复方案]] - 数据安全
- [[03-方法卡/数据库/MySQL灾难恢复演练\|MySQL灾难恢复演练]] - 容灾演练


## 13. 参考资源
- [MySQL 高可用官方文档](https://dev.mysql.com/doc/refman/8.0/en/ha-overview.html)
- [MHA 官方文档](https://github.com/yoshinorim/mha4mysql-manager)

