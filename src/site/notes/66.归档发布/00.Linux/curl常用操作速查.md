---
{"dg-publish":true,"permalink":"/66.归档发布/00.Linux/curl常用操作速查/","dg-note-properties":{"时间":"2026-03-21"}}
---

#linux #运维 #工具

```ad-summary
title: 总结

- curl 是 Linux 下最常用的 HTTP 调试工具，支持 GET/POST/PUT/DELETE 等所有方法
- `-X` 指定请求方法，`-H` 加请求头，`-d` 传请求体，`-o` 保存响应到文件
- `-v` 查看完整请求/响应头，排查问题必备；`-s` 静默模式，脚本里常用
- `--connect-timeout` 和 `--max-time` 控制超时，生产脚本里一定要加
```

## 1. GET 请求

```bash
# 基础 GET
curl http://localhost:8080/api/user

# 带查询参数
curl "http://localhost:8080/api/user?id=123&name=test"

# 带请求头（如 Token 鉴权）
curl -H "Authorization: Bearer your_token" \
     -H "Accept: application/json" \
     http://localhost:8080/api/user/123

# 查看响应头
curl -I http://localhost:8080/api/user

# 查看完整请求和响应（排查问题用）
curl -v http://localhost:8080/api/user
```

## 2. POST 请求

### 2.1 JSON 请求体

```bash
# 直接传 JSON 字符串
curl -X POST http://localhost:8080/api/user \
     -H "Content-Type: application/json" \
     -d '{"name":"张三","age":25}'

# 变量拼接 JSON（脚本中常用）
tenantId="t001"
gatewayIp="192.168.1.100"
JSON='{"code":"'${tenantId}'","version":"3.4.1"}'
RESULT=$(curl -s -X POST http://${gatewayIp}:8200/tenant/upgrade \
     -H "Content-Type: application/json" \
     -d "${JSON}")
echo "$RESULT"

# 从文件读取请求体
curl -X POST http://localhost:8080/api/user \
     -H "Content-Type: application/json" \
     -d @request.json
```

### 2.2 表单提交

```bash
# application/x-www-form-urlencoded
curl -X POST http://localhost:8080/login \
     -d "username=admin&password=123456"

# multipart/form-data（上传文件）
curl -X POST http://localhost:8080/upload \
     -F "file=@/path/to/file.txt" \
     -F "description=测试文件"
```

## 3. PUT / PATCH / DELETE

```bash
# PUT 更新资源
curl -X PUT http://localhost:8080/api/user/123 \
     -H "Content-Type: application/json" \
     -d '{"name":"李四","age":30}'

# PATCH 部分更新
curl -X PATCH http://localhost:8080/api/user/123 \
     -H "Content-Type: application/json" \
     -d '{"age":31}'

# DELETE
curl -X DELETE http://localhost:8080/api/user/123 \
     -H "Authorization: Bearer your_token"
```

## 4. 常用参数

| 参数 | 说明 |
|---|---|
| `-X METHOD` | 指定请求方法（GET/POST/PUT/DELETE 等） |
| `-H "Key: Value"` | 添加请求头，可多次使用 |
| `-d "data"` | 请求体，字符串或 `@文件路径` |
| `-F "key=value"` | 表单字段，`@路径` 表示上传文件 |
| `-o file` | 响应内容保存到文件 |
| `-O` | 用 URL 中的文件名保存响应 |
| `-v` | 显示完整请求/响应头，调试用 |
| `-s` | 静默模式，不显示进度条，脚本中常用 |
| `-i` | 响应中包含响应头 |
| `-I` | 只获取响应头（HEAD 请求） |
| `-L` | 跟随重定向 |
| `-u user:pass` | HTTP Basic 认证 |
| `-k` | 跳过 SSL 证书验证（测试环境用） |
| `--connect-timeout N` | 连接超时秒数 |
| `--max-time N` | 整个请求最大耗时秒数 |

## 5. 实用场景

### 5.1 下载文件

```bash
# 下载并保存为指定文件名
curl -o output.zip http://example.com/file.zip

# 用 URL 中的文件名保存，显示进度
curl -O http://example.com/file.zip

# 断点续传
curl -C - -O http://example.com/largefile.zip
```

### 5.2 HTTPS 与证书

```bash
# 跳过证书验证（测试环境）
curl -k https://localhost:8443/api/test

# 指定 CA 证书
curl --cacert /path/to/ca.crt https://example.com/api

# 双向认证（mTLS）
curl --cert /path/to/client.crt \
     --key /path/to/client.key \
     https://example.com/api
```

### 5.3 设置超时（生产脚本必加）

```bash
curl --connect-timeout 5 \
     --max-time 30 \
     -s -X POST http://localhost:8080/api/task \
     -H "Content-Type: application/json" \
     -d '{"taskId":"123"}' \
     >> /var/log/task.log 2>&1
```

### 5.4 响应状态码判断

```bash
#!/bin/bash
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health)

if [ "$HTTP_CODE" -eq 200 ]; then
    echo "服务正常"
else
    echo "服务异常，状态码: $HTTP_CODE"
fi
```

### 5.5 Cookie 操作

```bash
# 保存 Cookie 到文件
curl -c cookies.txt http://localhost:8080/login \
     -d "username=admin&password=123456"

# 携带 Cookie 发请求
curl -b cookies.txt http://localhost:8080/api/user

# 直接传 Cookie 字符串
curl -H "Cookie: session=abc123; token=xyz" \
     http://localhost:8080/api/user
```
