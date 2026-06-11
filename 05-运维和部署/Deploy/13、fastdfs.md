# docker 安装

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)

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
