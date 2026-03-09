---
{"dg-publish":true,"permalink":"/66.归档发布/00.Linux/SSH免密登录/"}
---

#linux
### 1. 生成密钥
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
### 2. copy 到目标机器
```bash
ssh-copy-id user@remote_host
```
