---
{"dg-publish":true,"permalink":"/66.归档发布/00.Linux/crontab定时任务使用指南/","dg-note-properties":{"时间":"2026-03-21"}}
---

#linux #运维 #最佳实践

```ad-summary
title: 总结

- crontab 是 Linux 定时任务工具，格式：分 时 日 月 周 命令
- 添加任务前先检查是否已存在，避免重复注册
- 常用场景：定期清理日志、数据库备份、磁盘监控告警、服务健康检查
- 脚本输出建议重定向到日志文件，方便排查执行情况
```

## 1. crontab 基础

### 1.1 时间格式

```
*  *  *  *  *  command
│  │  │  │  │
│  │  │  │  └── 星期（0-7，0 和 7 都表示周日）
│  │  │  └───── 月份（1-12）
│  │  └──────── 日期（1-31）
│  └─────────── 小时（0-23）
└────────────── 分钟（0-59）
```

常用示例：

```bash
0 1 * * *        # 每天凌晨 1 点
0 */6 * * *      # 每 6 小时执行一次
0 1 * * 1        # 每周一凌晨 1 点
0 1 1 * *        # 每月 1 号凌晨 1 点
*/5 * * * *      # 每 5 分钟执行一次
```

### 1.2 常用命令

```bash
crontab -l          # 查看当前用户的定时任务
crontab -e          # 编辑定时任务
crontab -r          # 删除所有定时任务（慎用）
crontab -u user -l  # 查看指定用户的定时任务
```

## 2. 定期清理日志

路径可按实际情况调整，脚本会检查任务是否已存在，避免重复注册。

```bash
#!/bin/sh
# 定期清理各服务日志，保留 180 天内的日志

cron_temp=$(mktemp)
crontab -l > "$cron_temp" 2>/dev/null

add_job_if_absent() {
    local tag="$1"
    local job="$2"
    if ! grep -qF "$tag" "$cron_temp"; then
        echo "添加任务: $tag"
        echo "$job" >> "$cron_temp"
    else
        echo "已存在: $tag，跳过"
    fi
}

# 应用日志，每天凌晨 1:00
add_job_if_absent "/home/logs/" \
    "0 1 * * * find /home/logs/* -mtime +180 -name '*.log' -exec rm -rf {} \;"

# Nginx 日志，每天凌晨 1:10
add_job_if_absent "/home/nginx/logs/" \
    "10 1 * * * find /home/nginx/logs/ -mtime +180 -name '*.log' -exec rm -rf {} \;"

# XXL-Job 日志，每天凌晨 1:20
add_job_if_absent "/data/applogs/xxl-job/" \
    "20 1 * * * find /data/applogs/xxl-job/* -mtime +180 -name '*.log' -exec rm -rf {} \;"

# Nacos 日志，每天凌晨 1:30
add_job_if_absent "/home/nacos/logs/" \
    "30 1 * * * find /home/nacos/logs/ -mtime +180 -name '*.log' -exec rm -rf {} \;"

crontab "$cron_temp"
rm -f "$cron_temp"

echo "当前定时任务："
crontab -l
```

## 3. 数据库定时备份

MySQL 每天凌晨 2 点备份，保留最近 7 天：

```bash
#!/bin/bash
# /opt/scripts/mysql_backup.sh

DB_USER="root"
DB_PASS="your_password"
DB_NAME="your_database"
BACKUP_DIR="/data/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz"

mkdir -p "$BACKUP_DIR"

# 备份并压缩
mysqldump -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" | gzip > "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "$(date): 备份成功 -> $BACKUP_FILE"
else
    echo "$(date): 备份失败！" >&2
fi

# 删除 7 天前的备份
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +7 -exec rm -f {} \;
```

注册定时任务：

```bash
# 每天凌晨 2 点执行，输出追加到日志
0 2 * * * /opt/scripts/mysql_backup.sh >> /var/log/mysql_backup.log 2>&1
```

## 4. 磁盘空间监控告警

磁盘使用率超过阈值时发送告警（需要配置好 `mail` 命令或替换为其他通知方式）：

```bash
#!/bin/bash
# /opt/scripts/disk_monitor.sh

THRESHOLD=85   # 告警阈值（%）
ALERT_EMAIL="ops@example.com"

df -h | grep -vE '^Filesystem|tmpfs|cdrom' | awk '{print $5 " " $6}' | while read usage mount; do
    usage_num=${usage%%%}  # 去掉百分号
    if [ "$usage_num" -ge "$THRESHOLD" ]; then
        msg="[告警] $(hostname) 磁盘 $mount 使用率已达 $usage，请及时处理！"
        echo "$msg" | mail -s "磁盘空间告警" "$ALERT_EMAIL"
        echo "$(date): $msg" >> /var/log/disk_monitor.log
    fi
done
```

注册定时任务：

```bash
# 每小时检查一次
0 * * * * /opt/scripts/disk_monitor.sh
```

## 5. 服务健康检查与自动重启

检查服务是否存活，挂了自动拉起：

```bash
#!/bin/bash
# /opt/scripts/service_watchdog.sh

SERVICE_NAME="your-app"
START_CMD="/opt/your-app/bin/start.sh"
LOG_FILE="/var/log/watchdog.log"

if ! systemctl is-active --quiet "$SERVICE_NAME"; then
    echo "$(date): $SERVICE_NAME 未运行，尝试重启..." >> "$LOG_FILE"
    systemctl start "$SERVICE_NAME"
    if systemctl is-active --quiet "$SERVICE_NAME"; then
        echo "$(date): $SERVICE_NAME 重启成功" >> "$LOG_FILE"
    else
        echo "$(date): $SERVICE_NAME 重启失败，请人工介入！" >> "$LOG_FILE"
    fi
fi
```

注册定时任务：

```bash
# 每 5 分钟检查一次
*/5 * * * * /opt/scripts/service_watchdog.sh
```

## 6. 定时清理临时文件

清理 `/tmp` 下超过 3 天未访问的文件：

```bash
# 每天凌晨 3 点清理
0 3 * * * find /tmp -atime +3 -type f -exec rm -f {} \; 2>/dev/null
```

## 7. 注意事项

**脚本要有执行权限**

```bash
chmod +x /opt/scripts/your_script.sh
```

**输出重定向，方便排查**

```bash
# 正常输出和错误输出都写入日志
0 1 * * * /opt/scripts/your_script.sh >> /var/log/cron_your_script.log 2>&1
```

**crontab 里用绝对路径**

crontab 执行时环境变量和交互式 shell 不同，`PATH` 可能不包含常用命令目录，脚本里的命令和文件路径都用绝对路径，或者在脚本开头显式设置：

```bash
#!/bin/bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**测试脚本再注册**

先手动执行脚本确认没问题，再注册到 crontab，避免静默失败。
