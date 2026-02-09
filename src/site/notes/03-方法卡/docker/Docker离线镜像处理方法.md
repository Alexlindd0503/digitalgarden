---
{"dg-publish":true,"permalink":"/03-方法卡/docker/Docker离线镜像处理方法/","tags":["离线部署","镜像管理","内网环境","方法卡","tech/docker"]}
---

## 1. 场景
内网环境无法直接 pull 镜像，通过 save/load 实现离线传输。

## 2. 操作流程

### 2.1. 导出镜像
```shell
# 联网环境导出
docker save java:8 -o java.tar
# 压缩
tar czvf java.tar.gz java.tar
# 另外一种压缩
gzip java.tar
```

### 2.2. 导入镜像  
```shell
# 内网环境导入
tar xzvf java.tar.gz

# 另外解压
gzip -d java.tar.gz

#导入镜像
docker load -i java.tar
```

## 3. 核心命令
- `docker save` - 导出镜像为 tar 文件
- `docker load` - 从 tar 文件导入镜像

