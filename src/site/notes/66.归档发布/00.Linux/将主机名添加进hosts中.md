---
{"dg-publish":true,"permalink":"/66.归档发布/00.Linux/将主机名添加进hosts中/","dg-note-properties":{"时间":"2026-03-21"}}
---

#linux #运维 #脚本

## 脚本

```bash
#!/bin/bash

# 需要 root 权限
if [[ $EUID -ne 0 ]]; then
    echo "请使用 root 权限执行此脚本" >&2
    exit 1
fi

HOSTNAME=$(hostname)

# 主机名不为空且不是 localhost 才处理
if [[ -n "$HOSTNAME" && "$HOSTNAME" != localhost* ]]; then
    # 正则中 IP 的点需要转义，避免误匹配
    if ! grep -Ew "127\.0\.0\.1.*\b${HOSTNAME}\b" /etc/hosts > /dev/null 2>&1; then
        echo "127.0.0.1 ${HOSTNAME}" >> /etc/hosts
        echo "已添加: 127.0.0.1 ${HOSTNAME}"
    else
        echo "已存在，无需重复添加: 127.0.0.1 ${HOSTNAME}"
    fi
fi
```

