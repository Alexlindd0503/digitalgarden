---
{"dg-publish":true,"permalink":"/03-方法卡/Docker工作目录迁移方法/","tags":["docker","存储管理","系统运维","磁盘迁移","方法卡","type/practice"]}
---

## 1. 场景
Docker 默认安装在 `/var/lib/docker`，当 `/var` 分区空间不足时，需要将 Docker 工作目录迁移到更大的磁盘分区。

## 2. 操作步骤

### 2.1. 停止 Docker 服务
```bash
systemctl stop docker
```

### 2.2. 创建新的工作目录
选择空间充足的分区（如 `/home`）创建新目录：
```bash
mkdir -p /home/docker/lib
```

### 2.3. 迁移数据
使用 rsync 迁移原有数据，保持文件属性和权限：
```bash
rsync -avz /var/lib/docker /home/docker/lib/
```

**参数说明：**
- `-a`: 归档模式，保留权限、时间戳等
- `-v`: 显示详细过程
- `-z`: 传输时压缩数据

### 2.4. 配置新路径
创建或编辑 Docker 服务配置文件：
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d/
sudo vi /etc/systemd/system/docker.service.d/devicemapper.conf
```

写入以下配置：
```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --graph=/home/docker/lib/docker
```

> **注意**: 第一行 `ExecStart=` 用于清空默认配置，第二行设置新路径。

### 2.5. 重启 Docker 服务
```bash
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
```

### 2.6. 验证迁移结果
检查 Docker 信息，确认新路径生效：
```bash
docker info | grep "Docker Root Dir"
```

验证原有镜像和容器是否正常：
```bash
docker images
docker ps -a
```

### 2.7. 清理旧数据
确认迁移成功后，删除原目录释放空间：
```bash
rm -rf /var/lib/docker
```

## 3. 注意事项
- 迁移前确保新分区有足够空间
- 迁移过程中不要启动 Docker 服务
- 建议先备份重要数据
- 验证无误后再删除原目录

