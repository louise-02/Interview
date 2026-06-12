# FastDFS 简介

[FastDFS GitHub](https://github.com/happyfish100/fastdfs) | [Wiki](https://github.com/happyfish100/fastdfs/wiki)

FastDFS 是国人开发的**分布式文件存储**系统，由 Tracker（调度）和 Storage（存储）组成，曾广泛用于图片、附件上传场景。

| 组件 | 说明 |
| ---- | ---- |
| Tracker | 调度节点，管理 Storage 分组 |
| Storage | 存储节点，保存文件 |

## 部署方式推荐

| 环境 | 推荐方式 | 说明 |
| ---- | -------- | ---- |
| **生产** | **原生部署（Tracker + Storage）** | 社区 Docker 镜像维护较弱，生产需充分验证 |
| 开发 / 测试 | Docker | 文档提供 compose，快速体验 |
| 新项目 | 评估 **MinIO** 等对象存储 | S3 兼容，生态更活跃，见 [docker/0、目录规划.md](./docker/0、目录规划.md) 待补充列表 |

---

# docker 安装（delron/fastdfs）

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)  
> Docker 偏生产不推荐 FastDFS；正式生产见下文 [FastDFS 配置与运维](#fastdfs-配置与运维)（原生部署）

## 1、创建目录

```bash
mkdir -p /data/docker/fastdfs/{tracker,storage}
```

## 2、启动

```bash
cd /path/to/Deploy/docker
docker compose -f fastdfs.yml up -d
```

或使用 docker run：

```bash
docker run -d \
  --name fastdfs-tracker \
  --network=host \
  -v /data/docker/fastdfs/tracker:/var/fdfs \
  delron/fastdfs tracker

docker run -d \
  --name fastdfs-storage \
  --network=host \
  -e TRACKER_SERVER=127.0.0.1:22122 \
  -e GROUP_NAME=group1 \
  -v /data/docker/fastdfs/storage:/var/fdfs \
  delron/fastdfs storage
```

## 3、离线传输镜像

```bash
docker pull --platform linux/amd64 delron/fastdfs
docker save -o fastdfs.tar delron/fastdfs
docker load -i fastdfs.tar
```

## 4、配置文件

```bash
# tracker
docker exec -it fastdfs-tracker bash
cat /etc/fdfs/client.conf

# storage
docker exec -it fastdfs-storage bash
cat /etc/fdfs/storage.conf | grep tracker

# 重启
docker restart fastdfs-tracker
docker restart fastdfs-storage
```

## 5、测试上传

```bash
docker exec -it fastdfs-tracker bash
echo "hello" > test.txt
fdfs_upload_file /etc/fdfs/client.conf test.txt

# 图片访问，8888 是 storage 端口
http://宿主机IP:8888/group1/M00/00/00/xxx.txt
```

# FastDFS 配置与运维

Docker 镜像仅适合开发测试。生产建议原生部署 Tracker + Storage 集群。

## 1、生产配置

```bash
# /etc/fdfs/tracker.conf
base_path=/data/fastdfs/tracker
http.server_port=8080

# /etc/fdfs/storage.conf
base_path=/data/fastdfs/storage
store_path0=/data/fastdfs/storage/data
tracker_server=tracker1:22122
tracker_server=tracker2:22122
```

## 2、集群与备份

- Tracker/Storage 分机部署，Storage 至少 2 台做冗余
- 上传目录挂载独立磁盘，`store_path` 空间监控告警
- 8888 端口通过 Nginx 反代，不要直接暴露 Storage
- 定期备份 `store_path` 数据目录
