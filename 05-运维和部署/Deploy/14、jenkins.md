# docker 安装

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)

## 1、创建目录

```bash
mkdir -p /data/docker/jenkins/data
```

## 2、启动

```bash
cd /path/to/Deploy/docker
docker compose -f jenkins.yml up -d
```

## 3、获取初始密码

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

浏览器访问 `http://宿主机IP:8080`，输入初始密码完成向导。

## 4、常用操作

```bash
docker compose -f docker/jenkins.yml ps
docker compose -f docker/jenkins.yml logs -f
docker compose -f docker/jenkins.yml down
```

> 注意：Jenkins 默认占用 8080 端口，与 Kafka UI 冲突，同时部署时需修改其中一个端口映射。
