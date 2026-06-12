# Redis 简介

[Redis 官网](https://redis.io) | [文档](https://redis.io/docs/)

Redis 是内存型 **KV 存储**，常用于缓存、Session、分布式锁、消息队列（Stream）等。

| 功能 | 说明 |
| ---- | ---- |
| 缓存 | 减轻数据库压力 |
| 持久化 | RDB / AOF 可选 |
| 主从 / 哨兵 | 高可用方案 |

常用端口：

| 端口 | 说明 |
| ---- | ---- |
| 6379 | 客户端连接 |

## 部署方式推荐

| 环境 | 推荐方式 | 说明 |
| ---- | -------- | ---- |
| **生产** | **yum 或编译** | 便于内存、持久化、maxmemory 等调优 |
| 容器化环境 | Docker | 可用，注意数据卷挂载与内存限制 |
| 开发 / 测试 | Docker | 快速搭建 |

---

# docker 安装（Redis 7.2）

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)  
> 偏生产检查清单见 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md)

## 1、创建目录

```bash
mkdir -p /data/docker/redis/{conf,data}
```

## 2、偏生产配置（挂载）

compose 已指定挂载 `redis.conf`，**首次 `up` 前**必须准备该文件：

```bash
cd /path/to/Deploy/docker
cp conf/redis/redis.conf /data/docker/redis/conf/
vi /data/docker/redis/conf/redis.conf   # 改 requirepass、maxmemory
```

`maxmemory`、`appendonly`、`rename-command` 等说明见下文 [Redis 配置与运维](#redis-配置与运维)。若设置了 `requirepass`，健康检查需改为 `redis-cli -a 密码 ping`。

## 3、启动

```bash
docker compose -f redis.yml up -d
```

## 4、测试

```bash
docker exec -it redis redis-cli -a '你的密码' ping
```

# yum 安装（Redis 7.2.4）

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

# 编译安装（Redis 7.2.4）

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
bind 0.0.0.0
protected-mode yes
requirepass 强密码
# 最大内存：专用节点约物理内存 60%～75%，混部约 40%～50%
maxmemory 4gb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
```

> `protected-mode yes` 时务必设置 `requirepass` 或 bind + 防火墙限制。

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

# Redis 配置与运维

## 1、安装目录

| 类型           | 路径                                         | 说明                                    |
| -------------- | -------------------------------------------- | --------------------------------------- |
| 📄 主程序       | `/usr/bin/redis-server`                      | Redis 服务端执行文件                    |
| 📄 客户端工具   | `/usr/bin/redis-cli`                         | 命令行客户端工具                        |
| 📁 配置文件     | `/etc/redis.conf` 或 `/etc/redis/redis.conf` | 主配置文件（具体路径依版本而定）        |
| 📁 服务控制文件 | `/usr/lib/systemd/system/redis.service`      | systemd 服务脚本（用于启动/停止 Redis） |
| 📁 数据目录     | `/var/lib/redis/`                            | 持久化数据文件（如 dump.rdb）           |
| 📁 日志文件     | `/var/log/redis/redis.log`                   | 日志文件路径（可在配置文件中更改）      |
| 📁 运行时文件   | `/var/run/redis/`                            | PID 文件、socket 等运行时文件           |

### 编译安装目录

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

## 2、生产配置

**maxmemory 怎么设？** 按机器**物理内存**粗算（不是只看 `free`）：

| 部署方式 | 建议占物理内存 | 说明 |
| -------- | -------------- | ---- |
| **专用 Redis 节点** | **60%～75%** | 单机只跑 Redis，余量给 OS + 持久化 fork |
| **与应用混部** | **40%～50%** | 同一台还跑 Java/Nginx 等，再压低 |
| 开启 AOF/RDB | 在上述基础上**偏保守** | fork 写盘时可能短暂多占内存（CoW），留 10%～20% 缓冲 |

示例：8GB 机器专用节点 → `maxmemory 5gb`（约 62%）；16GB → `maxmemory 10gb`～`12gb`（约 62%～75%）。下文 `4gb` 仅作 8GB 混部场景的写法示例，按实际内存替换。

Docker / yum / 编译共用以下关键项（路径按实际安装调整）：

```bash
# 监听所有网卡（生产配合防火墙限制来源）
bind 0.0.0.0
# 无密码且未 bind 127.0.0.1 时拒绝外网（需配合 requirepass）
protected-mode yes
# 客户端认证密码
requirepass 强密码
# 最大内存：专用节点约物理内存 60%～75%，混部约 40%～50%（16GB 专用可设 10gb～12gb）
maxmemory 4gb
# 内存满时淘汰最近最少使用的 key（纯缓存场景）
maxmemory-policy allkeys-lru
# 开启 AOF 持久化，重启少丢数据
appendonly yes
# 每秒 fsync 一次，平衡性能与安全
appendfsync everysec
# RDB：900 秒内至少 1 次写入则快照
save 900 1
save 300 10
save 60 10000
# 禁用 FLUSHALL / FLUSHDB / CONFIG，防误删与运行时改配置
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
```

## 3、系统参数

Redis 持久化 fork 时依赖系统内存策略，通用模板见 [0、readme.md](./0、readme.md#系统参数生产通用)。

```bash
# vm.overcommit_memory = 1
# 关闭透明大页 THP
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled

# /etc/security/limits.conf
redis soft nofile 65536
redis hard nofile 65536
```

## 4、检查与备份

```bash
redis-cli -a 强密码 ping
redis-cli -a 强密码 info memory
redis-cli -a 强密码 info persistence

ls -lh /data/redis/current/data/   # 备份 AOF/RDB
```

## 5、防火墙

```bash
sudo firewall-cmd --permanent --add-port=6379/tcp
sudo firewall-cmd --reload
```