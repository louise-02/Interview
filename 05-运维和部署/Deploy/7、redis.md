# docker 安装

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)

## 1、创建目录

```bash
mkdir -p /data/docker/redis/{conf,data}
```

## 2、准备配置

```bash
# 从镜像中拷贝默认配置再修改
docker run --rm redis:7.2 cat /usr/local/etc/redis/redis.conf > /data/docker/redis/conf/redis.conf
vi /data/docker/redis/conf/redis.conf
# 修改 protected-mode no，注释 bind 127.0.0.1
```

## 3、启动

```bash
cd /path/to/Deploy/docker
docker compose -f redis.yml up -d
```

## 4、测试

```bash
docker exec -it redis redis-cli ping
```

# yum 安装

## 1、安装

配置 Redis 的官方仓库

```bash
sudo vi /etc/yum.repos.d/redis.repo
```

然后添加以下内容

```bash
[remi-redis]
name=Remi's RPM repository for Redis
baseurl=https://rpms.remirepo.net/enterprise/7/remi/x86_64/
enabled=1
gpgcheck=1
gpgkey=https://rpms.remirepo.net/RPM-GPG-KEY-remi
```

安装 Redis

```bash
sudo yum install redis
# 指定版本
sudo yum install redis-7.2.4
```

启动 Redis 服务

```bash
sudo systemctl start redis
sudo systemctl enable redis
```

修改配置

```bash
protected-mode no
注释掉 bind
```

## 2、卸载

停止 Redis 服务

```bash
sudo systemctl stop redis
```

禁用 Redis 自动启动

```bash
sudo systemctl disable redis
```

卸载 Redis

```bash
sudo yum remove redis
```

删除 Redis 配置和数据目录（如果需要）

```bash
sudo rm -rf /etc/redis /var/lib/redis /var/log/redis
```

# 编译安装

## 1、安装

安装依赖

```bash
sudo yum install -y gcc make jemalloc-devel tcl
```

下载并编译 Redis 7.2.4

```bash
mkdir -p /opt/src && cd /opt/src
wget https://download.redis.io/releases/redis-7.2.4.tar.gz
tar xzf redis-7.2.4.tar.gz
cd redis-7.2.4
make && make install PREFIX=/opt/redis/7.2.4
```

创建目录和软链接

```bash
sudo mkdir -p /data/redis/7.2.4/{conf,data,logs,run}

sudo useradd -r -s /sbin/nologin redis
sudo chown -R redis:redis /data/redis/7.2.4
sudo chmod 750 /data/redis/7.2.4

# 创建版本软链接
sudo ln -sfn /opt/redis/7.2.4 /opt/redis/current
sudo ln -sfn /data/redis/7.2.4 /data/redis/current
```

## 2、配置文件

```bash
sudo cp /opt/src/redis-7.2.4/redis.conf /data/redis/current/conf/redis.conf
sudo chown redis:redis /data/redis/current/conf/redis.conf
sudo vi /data/redis/current/conf/redis.conf
```

修改这些关键配置项

```bash
dir /data/redis/current/data
logfile /data/redis/current/logs/redis.log
pidfile /data/redis/current/run/redis.pid
protected-mode no
注释掉 bind
```

## 3、添加服务

添加 systemd 服务文件

```
sudo tee /etc/systemd/system/redis.service > /dev/null <<EOF
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
User=redis
Group=redis
ExecStart=/opt/redis/current/bin/redis-server /data/redis/current/conf/redis.conf
ExecStop=/opt/redis/current/bin/redis-cli shutdown
Restart=always
LimitNOFILE=10032
PIDFile=/data/redis/current/run/redis.pid

[Install]
WantedBy=multi-user.target
EOF
```

## 4、启停命令

启动 Redis 服务并设置开机启动

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable redis
sudo systemctl start redis
sudo systemctl status redis
```

验证日志是否写入

```
tail -f /data/redis/current/logs/redis.log
```

## 5、卸载

停止 Redis 服务

```bash
sudo systemctl stop redis
sudo systemctl disable redis
```

删除 Redis 二进制文件

```bash
sudo rm -rf /opt/redis/7.2.4
```

删除 Redis 配置文件、数据文件和日志

```bash
sudo rm -rf /data/redis/7.2.4
```

删除 Redis systemd 服务

```bash
sudo rm -f /etc/systemd/system/redis.service
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
```

检查是否删除干净

```bash
which redis-server
redis-server --version
```

## 6、切换版本

```bash
sudo ln -sfn /opt/redis/7.2.5 /opt/redis/current
sudo ln -sfn /data/redis/7.2.5 /data/redis/current

# 然后重载服务
sudo systemctl restart redis
```

# redis

## 1、yum 安装目录

| 类型           | 路径                                         | 说明                                    |
| -------------- | -------------------------------------------- | --------------------------------------- |
| 📄 主程序       | `/usr/bin/redis-server`                      | Redis 服务端执行文件                    |
| 📄 客户端工具   | `/usr/bin/redis-cli`                         | 命令行客户端工具                        |
| 📁 配置文件     | `/etc/redis.conf` 或 `/etc/redis/redis.conf` | 主配置文件（具体路径依版本而定）        |
| 📁 服务控制文件 | `/usr/lib/systemd/system/redis.service`      | systemd 服务脚本（用于启动/停止 Redis） |
| 📁 数据目录     | `/var/lib/redis/`                            | 持久化数据文件（如 dump.rdb）           |
| 📁 日志文件     | `/var/log/redis/redis.log`                   | 日志文件路径（可在配置文件中更改）      |
| 📁 运行时文件   | `/var/run/redis/`                            | PID 文件、socket 等运行时文件           |

## 2、编译安装目录

如果配置了 `--prefix=/opt/redis` ，Redis 的安装目录会被集中在 `/opt/redis` 下，这样可以避免对系统其他目录的污染。

```bash
make
make PREFIX=/opt/redis install
```

| 类型         | 路径                          | 说明                               |
| ------------ | ----------------------------- | ---------------------------------- |
| 📄 主程序     | `/opt/redis/bin/redis-server` | Redis 服务端执行文件               |
| 📄 客户端工具 | `/opt/redis/bin/redis-cli`    | 命令行客户端工具                   |
| 📁 配置文件   | `/opt/redis/etc/redis.conf`   | 主配置文件（具体路径依版本而定）   |
| 📁 数据目录   | `/opt/redis/data/`            | 持久化数据文件（如 dump.rdb）      |
| 📁 日志文件   | `/opt/redis/logs/`            | 日志文件路径（可在配置文件中更改） |