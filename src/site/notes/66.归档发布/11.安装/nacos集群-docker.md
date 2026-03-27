---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/nacos集群-docker/","dg-note-properties":{"时间":"2026-03-27"}}
---

#nacos #docker #集群部署 #微服务

```ad-summary
title: 总结

- Nacos 2.x 集群最少要 3 台机器，单机或双节点跑不起来
- 用 host 网络模式最省事，节点间通讯天然解决，不用额外配容器网络
- Nacos 2.x 开了鉴权，客户端连接必须带账密，不然直接报错
- gRPC 端口是主端口 +1000，Nginx 的 stream 模块得配上去
```

## 1. 环境说明

Nacos 版本用的 2.2.3，集群模式最少 3 台机器，这里用的三台 IP 分别是：

- 192.168.50.9
- 192.168.50.18
- 192.168.50.245

假设 Docker 环境已经装好了，最终部署结构如下：

![attachment/8d3164e934cdd9a2a82e8ac50fe0c2f3_MD5.png](/img/user/attachment/8d3164e934cdd9a2a82e8ac50fe0c2f3_MD5.png)

## 2. 怎么部署？

### 2.1 先建库

Nacos 的集群状态要持久化，得先把官方提供的 `mysql-schema.sql` 导入到任意一个 MySQL 里。

> 这个脚本在 Nacos 的安装包 `conf` 目录下能找到，导入后会创建 `config_info`、`his_config_info` 等一堆表。

### 2.2 三台机器分别启动

每台机器执行下面的命令，注意改几个地方：

- `NACOS_SERVERS`：改成你实际的三台 IP
- `MYSQL_SERVICE_*`：改成你的 MySQL 连接信息
- 鉴权相关的 key、value、token：改成你自己的，别用默认值

```bash
docker run -d \
  -v /home/nacos/logs:/home/nacos/logs \
  -v /etc/hosts:/etc/hosts \
  -e MODE=cluster \
  -e NACOS_SERVERS="192.168.50.9:8848 192.168.50.18:8848 192.168.50.245:8848" \
  -e SPRING_DATASOURCE_PLATFORM=mysql \
  -e NACOS_AUTH_ENABLE=true \
  -e NACOS_AUTH_IDENTITY_KEY=linxxx \
  -e NACOS_AUTH_IDENTITY_VALUE=linxxx \
  -e NACOS_AUTH_TOKEN=linxxxlinxxxlinxxxlinxxxlinxxxlinxxxlinxxxlinxxxlinxxxlinxxx \
  -e MYSQL_SERVICE_HOST=192.168.50.123 \
  -e MYSQL_SERVICE_PORT=23306 \
  -e MYSQL_SERVICE_DB_NAME=nacos_config \
  -e MYSQL_SERVICE_USER=root \
  -e MYSQL_SERVICE_PASSWORD=3er4#ER$ \
  --restart unless-stopped \
  --net=host \
  --name nacos-cluster \
  nacos/nacos-server:v2.2.3
```

**网络模式用的 host**，这样三台 Nacos 之间通讯天然就通了，不用折腾容器网络。缺点就是端口直接占用宿主机的，如果宿主机上有其他服务用了 8848 就会冲突。

### 2.3 看日志验证

三台都启动后，等一会儿让集群选举完成，然后看日志：

```bash
docker logs nacos-cluster
```

看到类似下面的日志就说明集群 OK 了：

![attachment/a1129608c9cef859825908c4099a10bf_MD5.png](/img/user/attachment/a1129608c9cef859825908c4099a10bf_MD5.png)

### 2.4 Nginx 怎么配？

Nacos 2.x 除了 HTTP 的 8848 端口，还有个 gRPC 端口（主端口 +1000，也就是 9848）。所以 Nginx 要配两个东西：

1. **stream 模块**：处理 gRPC 的 TCP 连接，监听 8800 端口
2. **http 模块**：处理 HTTP 请求，监听 7800 端口

```nginx
stream {
  upstream nacos-cluster {
    server 192.168.50.9:9848;
    server 192.168.50.18:9848;
    server 192.168.50.245:9848;
  }
  server {
    listen 8800;
    proxy_connect_timeout 20s;
    proxy_timeout 5m;
    proxy_pass nacos-cluster;
  }
}

server {
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  listen 7800;
  upstream nacosserver {
    server 192.168.50.9:8848;
    server 192.168.50.18:8848;
    server 192.168.50.245:8848;
  }

  location /nacos/ {
    proxy_pass http://nacosserver/nacos/;
  }
}
```

> 8800 端口的 TCP 监听就是给 gRPC 用的。Nacos 的规则是 HTTP 端口 +1000 = gRPC 端口，所以 HTTP 起在 7800，gRPC 就是 8800，刚好对上。

## 3. 踩坑记录

Nacos 2.2.3 默认开了鉴权，客户端连接的时候**必须带账密**，不然直接连接失败，报错也不太明显，容易踩坑：

![attachment/4708d19e4272ed9c6c36adb926b63339_MD5.png](/img/user/attachment/4708d19e4272ed9c6c36adb926b63339_MD5.png)

连接参数里加上用户名密码就行，别漏了。
