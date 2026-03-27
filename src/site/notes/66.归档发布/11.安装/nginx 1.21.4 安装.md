---
{"dg-publish":true,"permalink":"/66.归档发布/11.安装/nginx 1.21.4 安装/","dg-note-properties":{"时间":"2026-03-27"}}
---

#nginx #web服务器 #反向代理 #运维

```ad-summary
title: 总结

- 源码编译需要先装依赖：gcc、pcre、zlib、openssl
- configure 之前先进入解压目录，指定安装路径方便后续管理
- 注册成 systemd 服务后才能用 systemctl 管理，开机自启也靠这个
- 防火墙记得开端口，不然外网访问不了
```

## 1. 安装依赖

```bash
yum -y install gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

## 2. 下载源码包

[官网下载页](https://nginx.org/en/download.html)

```bash
wget https://nginx.org/download/nginx-1.21.4.tar.gz
```

## 3. 编译安装

```bash
# 解压
tar -zxvf nginx-1.21.4.tar.gz
cd nginx-1.21.4

# 编译安装（指定安装路径）
./configure --prefix=/usr/local/nginx
make && make install
```

> 注意是 `nginx-1.21.4.tar.gz`，不是 `nginx.1.21.4.tar.gz`。另外 `&&` 表示前一条成功才执行后一条，单个 `&` 是后台运行，别搞混了。

## 4. 常用命令

```bash
# 启动
/usr/local/nginx/sbin/nginx

# 快速停止
/usr/local/nginx/sbin/nginx -s stop

# 优雅关闭（处理完当前请求再停）
/usr/local/nginx/sbin/nginx -s quit

# 重新加载配置
/usr/local/nginx/sbin/nginx -s reload
```

## 5. 注册系统服务

```bash
vi /usr/lib/systemd/system/nginx.service
```

```ini
[Unit]
Description=Nginx - HTTP Server
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable nginx
systemctl start nginx
systemctl status nginx
```

## 6. 开放端口

如果开了防火墙，记得放行 80 端口：

```bash
# 放行端口
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --reload

# 或者直接关防火墙（不推荐生产环境这么做）
systemctl stop firewalld
```

装完访问 `http://你的IP`，看到 Nginx 欢迎页就成功了。
