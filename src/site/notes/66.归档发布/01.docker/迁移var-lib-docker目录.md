---
{"dg-publish":true,"permalink":"/66.归档发布/01.docker/迁移var-lib-docker目录/","dg-note-properties":{}}
---

#docker

```ad-summary
title: 总结

- `/var` 磁盘不够时，用 rsync 把 `/var/lib/docker` 迁移到大盘，再改 dockerd 启动参数指向新路径
```

## 1. 停止 docker

```bash
systemctl stop docker
```

## 2. 创建新目录

```bash
mkdir -p /home/docker/lib
```

## 3. 迁移数据

```bash
rsync -avz /var/lib/docker /home/docker/lib/
```

## 4. 修改 dockerd 启动参数

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d/
sudo vi /etc/systemd/system/docker.service.d/devicemapper.conf
```

写入：

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --graph=/home/docker/lib/docker
```

## 5. 重启并验证

```bash
systemctl daemon-reload
systemctl restart docker
systemctl enable docker

docker info
```

确认原来的镜像还在，就可以删掉 `/var/lib/docker` 原目录了。
