---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/RocketMQ集群dlegger部署/","dg-note-properties":{"时间":"2026-03-26"}}
---

#rocketmq #消息队列 #dledger #集群部署 #安装

```ad-summary
title: 总结

- 3 台机器部署 3 个 NameServer + 9 个 Broker（3 组，每组 3 个），组内自动选举 master
- 配置文件主要改三个地方：brokerIP1、namesrvAddr、dLegerPeers
- 每台机器跑 3 个 Broker 容器，对应的配置文件不同
- 部署完在 console 看到所有 Broker 都是在线状态就说明成功了
```

## 1. 架构说明

RocketMQ 用 Dledger 模式部署，3 台机器，每台跑 1 个 NameServer + 3 个 Broker，共 9 个 Broker 分 3 组，组内通过 Raft 协议自动选 master。单机挂了会自动重新选举，不影响业务。

三台机器 IP：`192.168.50.18`、`192.168.50.245`、`192.168.50.9`

| 机器 | IP | 部署服务 | Broker 配置文件 |
|------|-----|---------|----------------|
| A | 192.168.50.18 | NameServer + Broker | broker-n 0、broker-n 5、broker-n 7 |
| B | 192.168.50.245 | NameServer + Broker | broker-n 2、broker-n 4、broker-n 6 |
| C | 192.168.50.9 | NameServer + Broker | broker-n 1、broker-n 3、broker-n 8 |

## 2. 配置文件处理

配置文件在 NAS 上，下载后需要改三个地方：

- `brokerIP1`：改成当前机器 IP
- `namesrvAddr`：三台机器的 NameServer 地址，用分号分隔
- `dLegerPeers`：组内三个 Broker 的地址，用分号分隔

> 建议用统一替换，别手动一个个改，容易漏。

把配置文件放到 `/home/rocketmq/config`：

```bash
mkdir -p /home/rocketmq/config
```

## 3. NameServer 部署

A、B、C 三台机器各执行一次：

```bash
docker run -d \
--restart=always \
--name rmqnamesrv \
--privileged=true \
-p 10111:9876 \
-v /etc/localtime:/etc/localtime \
-v /home/rocketmq/logs:/home/rocketmq/logs \
-v /home/rocketmq/store:/home/rocketmq/store \
-e "MAX_POSSIBLE_HEAP=100000000" \
-e "JAVA_OPT_EXT=-server -Xms1g -Xmx1g" \
apache/rocketmq:4.9.7 \
sh mqnamesrv
```

- `-p 10111:9876`：把容器内 9876 端口映射到宿主机 10111
- `JAVA_OPT_EXT`：NameServer JVM 堆内存，1 G 够用
- `MAX_POSSIBLE_HEAP`：容器内存上限，设了 `JAVA_OPT_EXT` 后这个参数会被覆盖，可以不加

## 4. Broker 部署

### 4.1 创建用户和目录

三台机器都执行，创建 rocketmq 用户：

```bash
groupadd rocketmq && useradd -g rocketmq rocketmq && groupmod -g 3000 rocketmq && usermod -u 3000 rocketmq
```

创建 9 个 Broker 的工作目录：

```bash
for ((i=0; i<=8; i++))
do
  mkdir -p /home/rocketmq/broker-n${i}/logs
  mkdir -p /home/rocketmq/broker-n${i}/store
done
```

授权：

```bash
chown -R rocketmq:rocketmq /home/rocketmq
```

### 4.2 启动 Broker 容器

每台机器跑 3 个 Broker 容器，对应的配置文件不同：

**A 机器（192.168.50.18）**：

```bash
sh install-n0.sh && sh install-n5.sh && sh install-n7.sh
```

**B 机器（192.168.50.245）**：

```bash
sh install-n2.sh && sh install-n4.sh && sh install-n6.sh
```

**C 机器（192.168.50.9）**：

```bash
sh install-n1.sh && sh install-n3.sh && sh install-n8.sh
```

> 安装脚本在 NAS 上，下载后放到对应机器执行即可。

## 5. Console 部署

改一下真实的 IP，执行：

```bash
docker run -d \
--name rmqconsole \
-v /etc/localtime:/etc/localtime:ro \
-e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.50.18:10111;192.168.50.245:10111;192.168.50.9:10111 -Dcom.rocketmq.sendMessageWithVIPChannel=false" \
-p 18080:8080 \
-t styletang/rocketmq-console-ng
```

访问 `http://IP:18080` 进入 Console 页面。

## 6. 验证

部署完成后在 Console 页面看到所有 Broker 都是在线状态，说明安装成功：

![4f5162fe18eb088567f375b3f18bd773 MD5](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/attachment/4f5162fe18eb088567f375b3f18bd773_MD5.png)

重点看：
- 每个 Broker Group 下有 1 个 master + 2 个 slave
- 状态都是 `ONLINE`
- Dledger 状态正常，没有异常告警
