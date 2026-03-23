---
{"dg-publish":true,"permalink":"/66.归档发布/00.Linux/rpm命令/","dg-note-properties":{"时间":"2026-03-21"}}
---

#linux #运维 #工具

```ad-summary
title: 总结

- rpm 是 RedHat 系（CentOS/RHEL/Fedora）的底层包管理工具，直接操作 .rpm 文件
- 常用四类操作：安装（-ivh）、升级（-Uvh）、卸载（-e）、查询（-q 系列）
- rpm 不自动处理依赖，缺依赖会报错；需要自动解决依赖用 yum/dnf
- 查包是否安装：`rpm -qa | grep 包名`；查文件属于哪个包：`rpm -qf 文件路径`
```

## 1. 安装

```bash
rpm -ivh 包全名.rpm
```

| 参数 | 说明 |
|---|---|
| `-i` | 安装（install） |
| `-v` | 显示详细信息（verbose） |
| `-h` | 显示安装进度条（hash） |

其他安装参数：

```bash
# 不检查依赖强制安装（不推荐，可能导致运行异常）
rpm -ivh --nodeps 包名.rpm

# 覆盖已存在的文件
rpm -ivh --replacefiles 包名.rpm

# 重复安装已安装的包
rpm -ivh --replacepkgs 包名.rpm

# 强制安装，等同于 --replacefiles + --replacepkgs
rpm -ivh --force 包名.rpm

# 测试安装（只检查不实际安装，用于验证依赖）
rpm -ivh --test 包名.rpm

# 指定安装路径
rpm -ivh --prefix /opt/myapp 包名.rpm
```

## 2. 升级

```bash
# 有旧版本则升级，没有则直接安装（常用）
rpm -Uvh 包名.rpm

# 只升级，没有安装过则不处理
rpm -Fvh 包名.rpm
```

`-U` 更常用，相当于"安装或升级"，不用关心是否已有旧版本。

## 3. 卸载

```bash
# 卸载（包名不需要带 .rpm 后缀）
rpm -e 包名

# 不检查依赖直接卸载（不推荐，可能导致其他软件崩溃）
rpm -e --nodeps 包名
```

卸载时用的是包名而不是文件名，可以先用 `rpm -qa | grep 关键词` 查到准确的包名再卸载。

## 4. 查询

```bash
# 查询所有已安装的包
rpm -qa

# 模糊搜索已安装的包
rpm -qa | grep java

# 查询某个包是否已安装
rpm -q 包名

# 查询包的详细信息（版本、描述、安装时间等）
rpm -qi 包名

# 查询未安装的 .rpm 文件信息
rpm -qip 包名.rpm

# 查询包安装了哪些文件
rpm -ql 包名

# 查询某个文件属于哪个包（排查文件来源时很有用）
rpm -qf /usr/bin/java

# 查询包的依赖关系
rpm -qR 包名

# 查询包的安装脚本
rpm -q --scripts 包名
```

## 5. 验证与校验

```bash
# 校验已安装包的文件完整性（检查文件是否被篡改）
rpm -V 包名

# 校验所有已安装包
rpm -Va

# 验证 .rpm 文件的 GPG 签名
rpm --checksig 包名.rpm

# 导入 GPG 公钥（安装前先导入，才能验证签名）
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

`rpm -V` 输出说明：

| 标记 | 含义 |
|---|---|
| `S` | 文件大小变化 |
| `M` | 权限或文件类型变化 |
| `5` | MD5 校验和变化 |
| `T` | 修改时间变化 |
| `U` | 所属用户变化 |
| `G` | 所属组变化 |

## 6. 常用组合场景

```bash
# 查找并卸载某个包
rpm -qa | grep nginx          # 先找到准确包名
rpm -e nginx-1.20.1-1.el7    # 再卸载

# 查看 java 命令来自哪个包
rpm -qf $(which java)

# 查看某个包装了哪些可执行文件
rpm -ql 包名 | grep bin

# 列出所有已安装包及版本，排序输出
rpm -qa --qf "%{NAME}-%{VERSION}\n" | sort
```

## 7. rpm vs yum/dnf

| | rpm | yum / dnf |
|---|---|---|
| 依赖处理 | 不自动处理，缺依赖报错 | 自动下载并安装依赖 |
| 安装来源 | 本地 .rpm 文件 | 本地文件或远程仓库 |
| 适用场景 | 离线安装、精确控制 | 日常安装、在线更新 |

有网络时优先用 `yum`/`dnf`，离线环境或需要精确控制版本时用 `rpm`。
