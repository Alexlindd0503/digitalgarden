---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/Nginx集群+keepalived/","dg-note-properties":{"时间":"2026-03-27"}}
---

#nginx #keepalived #高可用 #docker

```ad-summary
title: 总结

- Nginx + Keepalived 实现主备高可用，VIP 自动漂移，客户端无感知
- 主备配置就三个地方不同：state、priority、unicast_peer
- 健康检查脚本通过 curl 检测 Nginx 端口，返回非 200 就 kill 掉触发漂移
- 用 unicast 单播模式，不依赖组播，跨网段也能用
```

## 1. 高可用原理

![attachment/7ea270b4f6befdbe0d934d210b3f3cb3_MD5.png](/img/user/attachment/7ea270b4f6befdbe0d934d210b3f3cb3_MD5.png)

两台 Nginx 一主一备，Keepalived 监控 Nginx 状态，主节点挂了 VIP 自动漂移到备节点。

## 2. 环境信息

| 组件 | 版本 |
|------|------|
| Nginx | 1.23.1 |
| Keepalived | osixia/keepalived:2.0.20 |
| 主节点 | 192.168.50.22 |
| 备节点 | 192.168.50.251 |
| VIP | 192.168.50.127 |

## 3. 怎么部署？

### 3.1 部署 Nginx

两台机器都执行：

```bash
docker run -d \
  --name nginx \
  --net=host \
  --restart unless-stopped \
  -v /home/nginx/conf.d:/etc/nginx/conf.d \
  -v /home/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
  -v /home/nginx/html:/home/nginx/html \
  -v /home/nginx/logs:/var/log/nginx \
  nginx:1.23.1
```

把自己的 `nginx.conf` 放到宿主机 `/home/nginx/conf/` 目录下就行。

### 3.2 配置 Keepalived

#### 准备配置文件

两台机器都创建 `/home/keepalived/` 目录，放两个文件。

**主节点** keepalived.conf：

```ini
global_defs {
}

vrrp_script check_nginx {
  script "/container/service/keepalived/assets/check_nginx.sh"
  interval 5
  timeout 2
  weight 50
  rise 3
  fall 3
  user root
}

vrrp_instance VI_1 {
  interface em2                    # 网卡名，改成你的（ip addr 查看）
  state MASTER                     # 主节点
  virtual_router_id 51
  priority 100                     # 主节点优先级高
  nopreempt

  unicast_peer {
    192.168.50.251                 # 备节点 IP
  }

  virtual_ipaddress {
    192.168.50.127                 # VIP
  }

  authentication {
    auth_type PASS
    auth_pass 3er4#ER$
  }

  track_script {
    check_nginx
  }

  notify "/container/service/keepalived/assets/notify.sh"
}
```

**备节点**改三个地方：

| 参数 | 主节点 | 备节点 |
|------|--------|--------|
| state | MASTER | BACKUP |
| priority | 100 | 90 |
| unicast_peer | 填备节点 IP | 填主节点 IP |

> 查网卡名：`ip addr`，找到你实际用的网卡。

#### 健康检查脚本

```bash
vi /home/keepalived/check_nginx.sh
```

```bash
#!/bin/bash
http_status=$(curl -sIL -w "%{http_code}\n" -o /dev/null http://127.0.0.1:7800)
if [ ${http_status} != 200 ]; then
    kill 1
fi
```

> 端口改成你实际 Nginx 监听的端口。

#### 启动 Keepalived

两台机器都执行：

```bash
chmod +x /home/keepalived/check_nginx.sh

docker run -d \
  --name keepalived \
  --net=host \
  --cap-add=NET_ADMIN \
  --privileged=true \
  --restart unless-stopped \
  -v /etc/localtime:/etc/localtime \
  -v /home/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf \
  -v /home/keepalived/check_nginx.sh:/container/service/keepalived/assets/check_nginx.sh \
  osixia/keepalived:2.0.20 --copy-service
```

### 3.3 验证

```bash
# 在两台机器上分别查看 VIP
ip addr | grep 192.168.50.127

# 从其他机器 ping VIP
ping 192.168.50.127
```

VIP 只会出现在主节点上。停掉主节点的 Nginx，VIP 会漂移到备节点。

## 4. 卸载 VIP

手动删除 VIP（一般不需要，Keepalived 会自动管理）：

```bash
ip addr del 192.168.50.127/32 dev em2
```
