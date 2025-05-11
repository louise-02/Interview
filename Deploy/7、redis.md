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
cd /usr/local/src
wget https://download.redis.io/releases/redis-7.2.4.tar.gz
tar xzf redis-7.2.4.tar.gz
cd redis-7.2.4
make && make install
```

创建配置目录和数据目录

```bash
sudo mkdir -p /etc/redis /var/lib/redis /var/log/redis /var/run/redis

sudo useradd -r -s /sbin/nologin redis
sudo chown redis:redis /etc/redis /var/lib/redis /var/log/redis /var/run/redis
sudo chmod 750 /etc/redis /var/lib/redis /var/log/redis /var/run/redis
```

复制并修改配置文件

```bash
sudo cp redis.conf /etc/redis/redis.conf
sudo chown redis:redis /etc/redis/redis.conf
sudo vi /etc/redis/redis.conf
```

修改这些关键配置项

```bash
dir /var/lib/redis
logfile /var/log/redis/redis.log
protected-mode no
pidfile /var/run/redis/redis.pid
注释掉 bind
```

添加 systemd 服务文件

```
sudo tee /etc/systemd/system/redis.service > /dev/null <<EOF
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
User=redis
Group=redis
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
Restart=always
LimitNOFILE=10032

[Install]
WantedBy=multi-user.target
EOF
```

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
tail -f /var/log/redis/redis.log
```

## 2、卸载

停止 Redis 服务

```bash
sudo systemctl stop redis
sudo systemctl disable redis
```

删除 Redis 二进制文件

```bash
sudo rm -f /usr/local/bin/redis-server
sudo rm -f /usr/local/bin/redis-cli
sudo rm -f /usr/local/bin/redis-benchmark
sudo rm -f /usr/local/bin/redis-check-rdb
sudo rm -f /usr/local/bin/redis-check-aof
```

删除 Redis 配置文件

```bash
sudo rm -f /etc/redis/redis.conf
sudo rm -rf /etc/redis/
```

删除 Redis 数据文件和日志

具体取决于 redis.conf

```bash
sudo rm -rf /var/lib/redis/
sudo rm -rf /var/log/redis/
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

| 类型         | 路径                          | 说明                               |
| ------------ | ----------------------------- | ---------------------------------- |
| 📄 主程序     | `/opt/redis/bin/redis-server` | Redis 服务端执行文件               |
| 📄 客户端工具 | `/opt/redis/bin/redis-cli`    | 命令行客户端工具                   |
| 📁 配置文件   | `/opt/redis/etc/redis.conf`   | 主配置文件（具体路径依版本而定）   |
| 📁 数据目录   | `/opt/redis/data/`            | 持久化数据文件（如 dump.rdb）      |
| 📁 日志文件   | `/opt/redis/logs/`            | 日志文件路径（可在配置文件中更改） |