---
{"dg-publish":true,"permalink":"/学习记录/docker/迁移var-lib-docker目录/"}
---

#### 0.背景
有时候应为 docker 安装的目录在 /var 下，本身因为磁盘分配的比较小，需要将 docker 工作目录进行迁移.

#### 1. 停止 docker

```shell
systemctl stop docker
```

#### 2. 创建新的 docker 目录

找一个比较大的盘。比如 home 目录

```shell
mkdir -p /home/docker/lib
```

#### 3. 迁移数据

迁移/var/lib/docker 目录下面的文件到 /home/docker/lib

```shell
rsync -avz /var/lib/docker /home/docker/lib/
```

#### 4. 配置映射

配置 /etc/systemd/system/docker. Service. D/devicemapper. Conf。查看 devicemapper. Conf 是否存在。如果不存在，就新建。

```shell
sudo mkdir -p /etc/systemd/system/docker.service.d/
sudo vi /etc/systemd/system/docker.service.d/devicemapper.conf
```

写入数据：

```shell
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd  --graph=/home/docker/lib/docker
```

#### 5. 重新加载 docker

```shell
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
```

#### 6. 验证与删除

```shell
docker info
```

查看原来的镜像还在，即可以删除原来的数据。
