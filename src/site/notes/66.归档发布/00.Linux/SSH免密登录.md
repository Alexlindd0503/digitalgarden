---
{"dg-publish":true,"permalink":"/66.归档发布/00.Linux/SSH免密登录/"}
---

#linux

```ad-summary
title: 总结

- 两步搞定：本机生成密钥对，然后把公钥推到目标机器
```

## 1. 生成密钥

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

## 2. 推公钥到目标机器

```bash
ssh-copy-id user@remote_host
```
