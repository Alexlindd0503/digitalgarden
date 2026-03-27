---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/minio集群安装/","dg-note-properties":{"时间":"2026-03-27"}}
---

#minio #分布式存储 #对象存储 #运维

```ad-summary
title: 总结

- Docker 单机部署有单点故障，分布式模式必须用 Linux 原生安装
- 纠删码模式至少要 N/2+1 个磁盘存活，双机四磁盘挂一台就全挂，至少三机六磁盘
- 启动脚本里要列出所有节点的所有磁盘，三台机器的脚本内容一样
- Nginx 除了代理 9000 端口，还要代理 10000 端口给 console 用
```

## 1. 为什么不用 Docker？

Docker 部署 MinIO 只能做到单机多磁盘，存在单点故障。要搞分布式存储，得用 Linux 原生安装。

## 2. 部署方案怎么选？

MinIO 用纠删码模式，要求存活磁盘数 ≥ N/2+1：

| 方案 | 磁盘数 | 挂一台还剩 | 能用？ |
|------|--------|-----------|--------|
| 双机四磁盘 | 4 | 2 | ❌ 不满足 N/2+1 |
| 三机六磁盘 | 6 | 4 | ✅ 满足 N/2+1=4 |

所以至少要 **三机六磁盘**。本文用的三台机器：

- 192.168.50.245
- 192.168.50.18
- 192.168.50.9

MinIO 版本：`2023-09-23T03:47:50Z`

## 3. 怎么部署？

### 3.1 创建存储目录

**245 机器**：

```bash
mkdir -p /home/minio/data1 /home/minio/data2
```

**18 机器**：

```bash
mkdir -p /home/minio/data3 /home/minio/data4
```

**9 机器**：

```bash
mkdir -p /home/minio/data5 /home/minio/data6
```

### 3.2 下载 MinIO 二进制文件

三台机器都要执行：

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio -O /home/minio/minio
chmod +x /home/minio/minio
```

### 3.3 编写启动脚本

三台机器的脚本内容**完全一样**，都要列出所有节点的所有磁盘：

```bash
vi /home/minio/run.sh
```

```bash
#!/bin/bash
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=YourStrongPassword123!
/home/minio/minio server \
  http://192.168.50.245/home/minio/data1 \
  http://192.168.50.245/home/minio/data2 \
  http://192.168.50.18/home/minio/data3 \
  http://192.168.50.18/home/minio/data4 \
  http://192.168.50.9/home/minio/data5 \
  http://192.168.50.9/home/minio/data6 \
  --console-address ":10000"
```

> 密码别用弱密码，上面只是示例，生产环境换成强密码。

### 3.4 注册系统服务

```bash
vi /usr/lib/systemd/system/minio.service
```

```ini
[Unit]
Description=MinIO Service
Documentation=https://docs.min.io/

[Service]
WorkingDirectory=/opt/
ExecStart=/home/minio/run.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3.5 赋权限、启动

```bash
chmod +x /home/minio/minio /home/minio/run.sh

systemctl daemon-reload
systemctl enable minio
systemctl start minio
systemctl status minio
```

> 三台机器**都启动后**，节点之间才会建立连接。如果启动顺序有先后，先启动的会等待，这是正常的。
>
> 启动异常的话，检查一下脚本路径和 minio 二进制文件的位置是否对应，改完 `systemctl restart minio` 重启。

## 4. Nginx 怎么配？

MinIO 有 API 和 Console 两个端口，都要代理出来：

| 端口 | 用途 |
|------|------|
| 9000 | API 端口，客户端上传下载用 |
| 10000 | Console 端口，管理界面 |

```nginx
upstream minio {
    server 192.168.50.18:9000;
    server 192.168.50.245:9000;
    server 192.168.50.9:9000;
}

upstream minio-console {
    server 192.168.50.18:10000;
    server 192.168.50.245:10000;
    server 192.168.50.9:10000;
}

server {
    listen 9000;
    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-For $remote_addr;
        client_body_buffer_size 10M;
        client_max_body_size 10G;
        proxy_buffers 1024 4k;
        proxy_read_timeout 300;
        proxy_next_upstream error timeout http_404;
        proxy_pass http://minio;
    }
}

server {
    listen 10000;
    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://minio-console;
    }
}
```

配完 `nginx -t` 检查语法，再 `nginx -s reload` 生效。访问 `http://你的IP:10000` 就能进管理界面了。
