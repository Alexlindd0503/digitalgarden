---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/MongoDB-7.0.16集群安装-docker模式/","dg-note-properties":{"时间":"2026-03-27"}}
---

#mongodb #集群部署 #docker #副本集

```ad-summary
title: 总结

- 副本集解决高可用问题，分片集群解决水平扩展问题，大部分场景副本集就够了
- 密钥文件是集群内部认证用的，三台机器都要有，权限必须是 400
- data 目录权限要改成 999:999，否则 MongoDB 容器启动会报错
- docker 用了 host 网络模式就别加 -p 端口映射，两者冲突
```

## 1. 副本集还是分片集群？

MongoDB 集群有两种模式：

| 模式 | 目标 | 什么时候用 |
|------|------|-----------|
| Replica Set（副本集） | 高可用、数据冗余 | 数据量不大，要保证服务不挂 |
| Cluster（分片集群） | 水平扩展 | 海量数据、高吞吐量 |

大部分业务用副本集就够了。本文用的三台机器：

- 172.20.107.240
- 172.20.107.241
- 172.20.107.242

MongoDB 版本：`7.0.16`

## 2. 怎么部署？

### 2.1 三台机器都创建目录

**每台机器**都要执行：

```bash
mkdir -p /home/midware/mongo/data /home/midware/mongo/conf /home/midware/mongo/log
chown -R 999:999 /home/midware/mongo/data
```

> data 目录权限必须改成 999:999，这是 MongoDB 容器内部用户的 UID/GID，不然启动会报权限错误。

### 2.2 生成并分发密钥文件

在 240 机器上生成密钥，然后分发到另外两台：

```bash
# 生成密钥
openssl rand -base64 756 > /home/midware/mongo/conf/mongo-keyfile

# 分发到另外两台
scp /home/midware/mongo/conf/mongo-keyfile root@172.20.107.241:/home/midware/mongo/conf/
scp /home/midware/mongo/conf/mongo-keyfile root@172.20.107.242:/home/midware/mongo/conf/
```

**三台机器**都执行权限设置：

```bash
chown 999:999 /home/midware/mongo/conf/mongo-keyfile
chmod 400 /home/midware/mongo/conf/mongo-keyfile
```

> 密钥文件权限必须是 400，MongoDB 要求只能被属主读取。

### 2.3 启动容器

三台机器分别执行：

```bash
docker run -d \
  --name mongo-rs \
  --net=host \
  -v /home/midware/mongo/data:/data/db \
  -v /home/midware/mongo/conf/mongo-keyfile:/etc/mongo-keyfile \
  -v /home/midware/mongo/log:/data/log \
  -v /home/midware/mongo/conf:/etc/mongo \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=YourStrongPassword123! \
  mongo:7.0.16-jammy \
  mongod --replSet rs0 --keyFile /etc/mongo-keyfile --bind_ip_all
```

> 用了 `--net=host` 就**不要加 `-p` 端口映射**，两者冲突。host 模式下容器直接用宿主机网络，端口自动生效。

### 2.4 初始化副本集

进入任意一台容器：

```bash
docker exec -it mongo-rs mongosh
```

```javascript
// 认证
use admin
db.auth("admin", "YourStrongPassword123!")

// 初始化副本集
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "172.20.107.240:27017" },
    { _id: 1, host: "172.20.107.241:27017" },
    { _id: 2, host: "172.20.107.242:27017" }
  ]
})
```

### 2.5 验证集群状态

```javascript
rs.status().members.forEach(m => printjson({
  name: m.name,
  stateStr: m.stateStr
}))
```

看到一主两从（1 个 PRIMARY + 2 个 SECONDARY）就说明集群搭好了。等几秒让选举完成，刚初始化时可能短暂显示 STARTUP 是正常的。
