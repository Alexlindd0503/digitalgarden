---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/redis-cluster 集群安装/","dg-note-properties":{"时间":"2026-03-26"}}
---

#redis #集群 #docker #安装

```ad-summary
title: 总结

- 3 台机器 6 节点，3 主 3 从，用 Docker 部署
- 配置文件只需要改 cluster-announce-ip 为对应机器 IP
- 创建集群时前 3 个节点自动成为 master，后 3 个成为 replica
- 注意：当前架构下主从在同一台机器，机器挂了主从一起挂
```

## 1. 架构说明

Redis 6.2.7，3 台机器部署 3 主 3 从 6 节点，用 Docker 跑。每台机器跑 2 个 Redis 容器。

| 机器 | IP | 端口 |
|------|-----|------|
| A | 192.168.50.9 | 6371、6372 |
| B | 192.168.50.18 | 6373、6374 |
| C | 192.168.50.245 | 6375、6376 |

> 按 `--cluster-replicas 1` 的规则，前 3 个节点（6371、6373、6375）会成为 master，后 3 个（6372、6374、6376）成为 replica。这样主从会在同一台机器上，机器挂了主从一起挂。如果要真正的高可用，需要交叉部署，比如 6371 的 replica 放到别的机器。

## 2. 配置文件

配置文件在 NAS 上，一共 6 个：`redis-6371.conf` ~ `redis-6376.conf`。

每个配置文件里的 `cluster-announce-ip` 要改成**当前部署机器的 IP**，其他不用动。

## 3. 启动节点

### 3.1 A 机器（192.168.50.9）

```bash
for port in $(seq 6371 6372); do
  mkdir -p /home/redis-cluster/${port}/conf
  cp -r redis-${port}.conf /home/redis-cluster/${port}/conf/redis.conf
  docker run -di --log-opt max-size=100m --log-opt max-file=3 \
    --restart always --name redis-${port} --net host \
    -v /home/redis-cluster/${port}/conf/redis.conf:/etc/redis/redis.conf \
    -v /home/redis-cluster/${port}/data:/data \
    redis:6.2.7 redis-server /etc/redis/redis.conf
done
```

### 3.2 B 机器（192.168.50.18）

```bash
for port in $(seq 6373 6374); do
  mkdir -p /home/redis-cluster/${port}/conf
  cp -r redis-${port}.conf /home/redis-cluster/${port}/conf/redis.conf
  docker run -di --log-opt max-size=100m --log-opt max-file=3 \
    --restart always --name redis-${port} --net host \
    -v /home/redis-cluster/${port}/conf/redis.conf:/etc/redis/redis.conf \
    -v /home/redis-cluster/${port}/data:/data \
    redis:6.2.7 redis-server /etc/redis/redis.conf
done
```

### 3.3 C 机器（192.168.50.245）

```bash
for port in $(seq 6375 6376); do
  mkdir -p /home/redis-cluster/${port}/conf
  cp -r redis-${port}.conf /home/redis-cluster/${port}/conf/redis.conf
  docker run -di --log-opt max-size=100m --log-opt max-file=3 \
    --restart always --name redis-${port} --net host \
    -v /home/redis-cluster/${port}/conf/redis.conf:/etc/redis/redis.conf \
    -v /home/redis-cluster/${port}/data:/data \
    redis:6.2.7 redis-server /etc/redis/redis.conf
done
```


## 4. 创建集群

在 A 机器上进入任意一个 Redis 容器：

```bash
docker exec -it redis-6371 bash
```

执行集群创建命令：

```bash
redis-cli -a 3er4#ER$ --cluster create \
  192.168.50.9:6371 \
  192.168.50.9:6372 \
  192.168.50.18:6373 \
  192.168.50.18:6374 \
  192.168.50.245:6375 \
  192.168.50.245:6376 \
  --cluster-replicas 1
```

看到确认提示后输入 `yes`，等待分片完成：

![68052fe8bd708c9a8bb92dc234f9e0fd MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/68052fe8bd708c9a8bb92dc234f9e0fd_MD5.png)

## 5. 验证

创建完成后，进入任意节点查看集群状态：

```bash
redis-cli -a 3er4#ER$ cluster info
redis-cli -a 3er4#ER$ cluster nodes
```

看到 `cluster_state:ok`、`cluster_slots_ok:16384` 就说明集群正常。

## 6. 注意事项

- **扩容缩容**：Redis Cluster 的扩容和缩容都需要手动干预，不能自动平衡。加节点后要手动执行 `redis-cli --cluster reshard` 迁移槽位。
- **槽位覆盖**：16384 个槽位必须全部分配，集群才处于 OK 状态。
