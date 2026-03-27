---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/Elasticsearch7.17.27集群安装-docker/","dg-note-properties":{"时间":"2026-03-27"}}
---

#elasticsearch #docker #集群部署 #搜索引擎

```ad-summary
title: 总结

- ES 部署前必须调系统参数：vm.max_map_count、文件描述符上限，不然起不来
- 集群间通信要证书认证，生成一份分发到所有节点
- 容器默认用 uid=1000，可能被其他服务占了，改用 2000 更安全
- JVM 堆内存别超机器内存的一半，ES 容器的 --memory 要大于堆内存
```

## 1. 部署前准备

涉及三台机器：

- 172.20.107.240
- 172.20.107.241
- 172.20.107.242

ES 版本：`7.17.27`

## 2. 系统参数调整

ES 对系统资源要求比较严格，**三台机器都要改**，否则启动会报错。

### 2.1 内核参数

```bash
vi /etc/sysctl.conf
```

```ini
# 文件描述符上限
fs.file-max=655360
# 进程内存映射区域上限，ES 强制要求至少 262144
vm.max_map_count=262144
```

```bash
sysctl -p
```

### 2.2 用户资源限制

```bash
vi /etc/security/limits.conf
```

```ini
# root 用户（如果以 root 运行）
root hard nofile 65536
root soft nofile 65536

# elasticsearch 用户（后面会创建）
elasticsearch hard nofile 65536
elasticsearch soft nofile 65536
elasticsearch hard nproc 65536
elasticsearch soft nproc 65536
elasticsearch hard memlock unlimited
elasticsearch soft memlock unlimited
```


## 3. 创建专用用户

ES 强制要求非 root 运行，容器默认用 uid=1000。但 1000 可能被其他服务占了，改用 2000 更安全。

**三台机器都执行**：

```bash
# 检查 uid 1000 是否已被占用
cat /etc/passwd | grep ":1000:"

# 创建 ES 专用用户组
groupadd -g 2000 elasticsearch
useradd -r -s /bin/false -u 2000 -g 2000 elasticsearch

# 设置目录权限
chown -R 2000:2000 /home/midware/elasticsearch
```

> 如果 1000 已经被其他容器用了就别动，强行改会导致那个服务挂掉。

## 4. 生成集群通信证书

集群节点之间通信需要 TLS 证书认证。

### 4.1 三台都创建证书目录

```bash
mkdir -p /home/midware/elasticsearch/config/certs
```

### 4.2 在一台机器上生成证书

```bash
docker run --rm \
  -v /home/midware/elasticsearch/config/certs:/certs \
  elasticsearch:7.17.27 \
  bin/elasticsearch-certutil cert -out /certs/elastic-certificates.p12 -pass ''
```

### 4.3 分发到其他节点

```bash
scp /home/midware/elasticsearch/config/certs/elastic-certificates.p12 \
  root@172.20.107.241:/home/midware/elasticsearch/config/certs/

scp /home/midware/elasticsearch/config/certs/elastic-certificates.p12 \
  root@172.20.107.242:/home/midware/elasticsearch/config/certs/
```

### 4.4 三台都设置证书权限

```bash
chmod 640 /home/midware/elasticsearch/config/certs/elastic-certificates.p12
chown 2000:2000 /home/midware/elasticsearch/config/certs/elastic-certificates.p12
```

## 5. 启动容器

以 240 为例，另外两台改 `network.host` 和 `node.name` 就行：

```bash
docker run -d \
  --name es-node1 \
  --network host \
  --cap-add IPC_LOCK \
  --ulimit memlock=-1:-1 \
  -u "2000:2000" \
  -e TAKE_FILE_OWNERSHIP=true \
  -e cluster.name=es-cluster \
  -e network.host=172.20.107.240 \
  -e node.name=node1 \
  -e node.master=true \
  -e node.data=true \
  -e discovery.seed_hosts=172.20.107.240,172.20.107.241,172.20.107.242 \
  -e cluster.initial_master_nodes=node1,node2,node3 \
  -e ES_JAVA_OPTS="-Xms2048m -Xmx2048m" \
  -e ELASTICSEARCH_USERNAME=elastic \
  -e ELASTIC_PASSWORD=YourStrongPassword123! \
  -e xpack.security.enabled=true \
  -e xpack.security.transport.ssl.enabled=true \
  -e xpack.security.transport.ssl.verification_mode=certificate \
  -e xpack.security.transport.ssl.keystore.path=certs/elastic-certificates.p12 \
  -e xpack.security.transport.ssl.truststore.path=certs/elastic-certificates.p12 \
  -e LC_ALL=C.UTF-8 \
  -e LANG=C.UTF-8 \
  -v "/home/midware/elasticsearch/config:/usr/share/elasticsearch/config" \
  -v "/home/midware/elasticsearch/data:/usr/share/elasticsearch/data" \
  -v "/home/midware/elasticsearch/plugins:/usr/share/elasticsearch/plugins" \
  -v "/home/midware/elasticsearch/logs:/usr/share/elasticsearch/logs" \
  --memory 4g \
  elasticsearch:7.17.27
```

> 内存配置经验：JVM 堆（`-Xms2048m -Xmx2048m`）设为容器内存限制（`--memory 4g`）的一半左右，留点给 OS 缓存和 Lucene 用。堆内存别超过机器总内存的 50%。

三台机器参数对比：

| 节点 | network.host | node.name |
|------|-------------|-----------|
| 240 | 172.20.107.240 | node1 |
| 241 | 172.20.107.241 | node2 |
| 242 | 172.20.107.242 | node3 |

## 6. 验证集群状态

等一两分钟让节点完成选举：

```bash
curl -u "elastic:YourStrongPassword123!" \
  http://172.20.107.240:9200/_cluster/health?pretty
```

看到 `"status" : "green"` 就说明集群健康，`"number_of_nodes" : 3` 确认三个节点都加入了。

![attachment/18495e5f21cf94320189f87d0e6d30cc_MD5.png](/img/user/attachment/18495e5f21cf94320189f87d0e6d30cc_MD5.png)

也可以浏览器访问 `http://172.20.107.240:9200` 看 Kibana 或其他管理界面：

![attachment/2ef25fbf9f8e955102f9a9b225d80d87_MD5.png](/img/user/attachment/2ef25fbf9f8e955102f9a9b225d80d87_MD5.png)
