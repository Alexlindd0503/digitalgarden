---
{"dg-publish":true,"permalink":"/66.归档发布/00.Linux/find&grep使用/"}
---

#linux

```ad-summary
title: 总结

- grep 搜文件内容，find 搜文件路径，两者组合用可以精准定位文件
- grep 常用 `-i`（忽略大小写）、`-n`（行号）、`-r`（递归）
- find 常用 `-name`、`-type`、`-mtime`、`-exec` 对结果执行命令
```

## 1. grep - 文本内容搜索

grep 用于在文件中搜索包含特定关键词的行。
**基本语法：**
```bash
grep [选项] 关键词 文件名
```
**常用选项：**
- `-i` 忽略大小写
- `-n` 显示行号
- `-c` 只显示匹配的行数
- `-v` 反向匹配（显示不包含关键词的行）
- `-r` 递归搜索目录下所有文件

**实用场景：**
```bash
# 在日志文件中查找错误信息
grep -i "error" app.log

# 查找配置文件中的端口设置，并显示行号
grep -n "port" config.conf

# 统计代码中 TODO 出现的次数
grep -c "TODO" *.js

# 在当前目录递归查找包含 "password" 的文件
grep -rn "password" .

# 查找不包含注释的配置行
grep -v "^#" nginx.conf
```

## 2. find - 文件查找

find 用于按文件名、大小、时间等条件查找文件。
**基本语法：**
```bash
find 路径 -选项 [动作]
```

**常用选项：**
- `-name` 按文件名查找（支持通配符）
- `-type` 按文件类型查找（f=文件，d=目录）
- `-size` 按文件大小查找（+1M 表示大于1M，-1M 表示小于1M）
- `-mtime` 按修改时间查找（-7 表示7天内，+7 表示7天前）
- `-user` 按文件所有者查找

**常用动作：**
- `-print` 输出结果（默认）
- `-exec 命令 {} \;` 对找到的文件执行命令
- `-delete` 删除找到的文件

**实用场景：**

```bash
# 查找当前目录下所有 .log 文件
find . -name "*.log"

# 查找大于 100M 的文件
find /home -type f -size +100M

# 查找 7 天内修改过的 Python 文件
find . -name "*.py" -mtime -7

# 查找空目录
find . -type d -empty

# 删除 30 天前的日志文件
find /var/log -name "*.log" -mtime +30 -delete

# 查找并批量修改文件权限
find . -name "*.sh" -exec chmod +x {} \;

# 查找大文件并显示详细信息
find . -type f -size +50M -exec ls -lh {} \;

# 清理临时文件
find /tmp -name "*.tmp" -mtime +1 -exec rm -f {} \;
```

**组合使用场景：**

```bash
# 查找包含特定内容的文件
find . -name "*.conf" -exec grep -l "database" {} \;

# 在最近修改的文件中搜索关键词
find . -mtime -1 -type f -exec grep -Hn "ERROR" {} \;

# 统计项目代码行数
find . -name "*.java" -exec wc -l {} \; | awk '{sum+=$1} END {print sum}'
```