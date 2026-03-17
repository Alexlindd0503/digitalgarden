---
{"dg-publish":true,"permalink":"/66.归档发布/00-0其他/nginx location与proxy_pass斜杠规则/"}
---

#nginx #最佳实践

```ad-summary
title: 总结

- `proxy_pass` 末尾**不加斜杠**：location 路径会拼接到转发地址后面
- `proxy_pass` 末尾**加斜杠**：location 路径被替换掉，只转发剩余部分
- `location /api` 与 `location /api/` 匹配范围不同，推荐用带斜杠版本
```

## 1. proxy_pass 不加斜杠：路径拼接

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

请求 `/api/user` → 转发到 `http://127.0.0.1:8080/api/user`

location 路径原样拼接到目标地址后面。

## 2. proxy_pass 加斜杠：路径替换

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

请求 `/api/user` → 转发到 `http://127.0.0.1:8080/user`

`/api/` 被替换掉，只把后面的 `user` 拼上去。

## 3. proxy_pass 对比

| 配置                              | 请求路径        | 实际转发                      |
| ------------------------------- | ----------- | ------------------------- |
| `proxy_pass http://backend`     | `/api/user` | `http://backend/api/user` |
| `proxy_pass http://backend/`    | `/api/user` | `http://backend/user`     |
| `proxy_pass http://backend/v2/` | `/api/user` | `http://backend/v2/user`  |

记忆方式：**加斜杠 = 替换前缀，不加斜杠 = 保留原路径**。

## 4. location /api 与 /api/ 的区别

### 匹配范围

- `location /api/` — 只匹配以 `/api/` 开头的请求（如 `/api/user`）
- `location /api` — 匹配 `/api` 本身，也匹配 `/api/user`，甚至 `/apiXXX`

### 路径拼接行为

**proxy_pass 不带斜杠时**，两者转发结果相同，但匹配范围不同：

```nginx
location /api {
    proxy_pass http://backend;
}
# /api/user → http://backend/api/user
# /apitest  → http://backend/apitest  ← 多匹配了！
```

**proxy_pass 带斜杠时**，差异更明显：

```nginx
location /api/ {
    proxy_pass http://backend/;
}
# /api/user → http://backend/user  ✓

location /api {
    proxy_pass http://backend/;
}
# /api/user → http://backend//user  ← 双斜杠！
```

### 结论

实际反代 API 时，统一用 `location /api/`：语义精确，不会误匹配 `/apiother`，路径替换行为也更可预期。
