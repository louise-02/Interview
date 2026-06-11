# docker 安装

## 下载镜像

```bash
docker pull delron/fastdfs

# 如果外网不能下载 在有网的机器操作
docker pull --platform linux/amd64 delron/fastdfs
docker save -o fastdfs.tar delron/fastdfs
# 在没网的机器加载
docker load -i fastdfs.tar
```

## 启动

目录规划

```bash
mkdir -p /wms/data/fastdfs/tracker
mkdir -p /wms/data/fastdfs/storage
```

docker 启动

```bash
docker run -d \
  --name fastdfs-tracker \
  --network=host \
  -v /wms/data/fastdfs/tracker:/var/fdfs \
  delron/fastdfs tracker


docker run -d \
  --name fastdfs-storage \
  --network=host \
  -e TRACKER_SERVER=10.52.67.157:22122 \
  -e GROUP_NAME=group1 \
  -v /wms/data/fastdfs/storage:/var/fdfs \
  delron/fastdfs storage
```

docker 编排

```yml
fastdfs-tracker:
  image: delron/fastdfs
  command:
    - tracker
  volumes:
    - /wms/data/fastdfs/tracker:/var/fdfs
  ports:
  - 22122:22122/tcp
fastdfs-storage:
  image: delron/fastdfs
  command: storage
  environment:
    TRACKER_SERVER: fastdfs-tracker:22122
    GROUP_NAME: group1
  volumes:
    - /wms/data/fastdfs/storage:/var/fdfs
  ports:
  - 8888:8888/tcp
  - 23000:23000/tcp
  depends_on:
    - fastdfs-tracker
```

## 配置文件

```bash
# tracker
docker exec -it fastdfs-tracker bash
cat /etc/fdfs/client.conf
tracker_server=192.168.0.197:22122

# storage
docker exec -it fastdfs-storage bash
cat /etc/fdfs/storage.conf | grep tracker
tracker_server=10.52.67.157:22122

# 重启
docker restart fastdfs-tracker
docker restart fastdfs-storage
```

## 测试上传

```bash
# 测试上传
docker exec -it fastdfs-tracker bash
echo "hello" > test.txt
fdfs_upload_file /etc/fdfs/client.conf test.txt

# 图片访问 这里的8888 是 storage 的
http://xxxx:8888/group1/M00/00/00/CjRDnWoUAVuATbgQAAAAD_xoRuc935.txt
```

