---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/keepalived 1.2.18 安装/","dg-note-properties":{"时间":"2026-03-27"}}
---

#keepalived #高可用 #运维

```ad-summary
title: 总结

- Keepalived 配合 LVS 做高可用，Nginx 挂了自动重启，重启失败通知运维
- 安装需要 gcc、openssl 依赖，源码编译三步走：configure、make、make install
- 配置成系统服务后用 systemctl 管理，记得 chkconfig 开机自启
- VIP 配置和健康检查脚本是核心，脚本要加执行权限
```

## 1. LVS 和 Nginx 负载均衡有什么区别？

| 对比项 | LVS | Nginx |
|--------|-----|-------|
| 层级 | 四层（基于 TCP/IP + 端口） | 七层（基于 HTTP 协议） |
| 角色 | 跨域管理 Nginx 集群 | 管理后端服务器集群 |
| 性能 | 高，扛得住大流量 | 相对低一些 |

简单说，LVS 管 Nginx，Nginx 管后端服务，一层套一层。

## 2. Keepalived 是干嘛的？

Keepalived 主要做两件事：

1. **VIP 漂移**：主节点挂了，VIP 自动漂移到备节点，客户端无感知
2. **健康检查**：监听 Nginx 是否存活，挂了自动重启；重启多次还失败，就发邮件通知运维

请求链路：**客户端 → LVS(VIP) → Nginx → 真实服务器**

## 3. 安装 Keepalived

> 本文用的 1.2.18 版本，比较老了，生产环境建议用 2.x 版本。安装步骤差不多，只是下载链接不同。

### 3.1 安装依赖

```bash
yum install -y gcc openssl openssl-devel
```

### 3.2 编译安装

```bash
wget http://www.keepalived.org/software/keepalived-1.2.18.tar.gz
tar -zxvf keepalived-1.2.18.tar.gz -C /usr/local
cd /usr/local/keepalived-1.2.18
./configure --prefix=/usr/local/keepalived
make && make install
```

## 4. 注册成系统服务

因为没用默认路径安装，需要手动配置一下：

```bash
mkdir /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/

ln -s /usr/local/sbin/keepalived /usr/sbin/
ln -s /usr/local/keepalived/sbin/keepalived /sbin/
chkconfig keepalived on
```

## 5. 常用命令

```bash
# 启动
systemctl start keepalived

# 停止
systemctl stop keepalived

# 查看状态（启动失败时看这里）
systemctl status keepalived
journalctl -xe
```

## 6. 配置 VIP

### 6.1 主节点配置

```bash
vi /etc/keepalived/keepalived.conf
```

```ini
vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33                # 网卡名，改成你自己的
    virtual_router_id 121          # 主备节点要一致
    mcast_src_ip 192.168.212.143   # 本机 IP
    priority 100                   # 优先级，主节点要比备节点高
    nopreempt                      # 防止恢复后抢占
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    track_script {
        chk_nginx
    }

    virtual_ipaddress {
        192.168.212.200            # VIP 地址
    }
}
```

**备节点**把 `state` 改成 `BACKUP`，`priority` 设低一点（比如 90），其他配置保持一致。

### 6.2 健康检查脚本

```bash
vi /etc/keepalived/nginx_check.sh
```

```bash
#!/bin/bash
A=$(ps -C nginx --no-header | wc -l)
if [ $A -eq 0 ]; then
    /usr/local/nginx/sbin/nginx
    sleep 2
    if [ $(ps -C nginx --no-header | wc -l) -eq 0 ]; then
        killall keepalived
    fi
fi
```

记得加执行权限：

```bash
chmod +x /etc/keepalived/nginx_check.sh
```

脚本逻辑：检测到 Nginx 没了就重启，等 2 秒再检查，还是挂的话就把 keepalived 杀掉，触发 VIP 漂移。
