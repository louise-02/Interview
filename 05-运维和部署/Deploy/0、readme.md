# 目录规划

## 服务一览

各服务文档开头均有**简介**与**部署方式推荐**，生产环境选型请先阅读对应章节。

| 序号 | 服务 | 文档 | 生产推荐 |
| ---- | ---- | ---- | -------- |
| 1 | ClickHouse | [1、clickhouse.md](./1、clickhouse.md) | yum |
| 2 | JDK | [2、jdk.md](./2、jdk.md) | yum / 二进制 |
| 3 | MySQL | [3、mysql.md](./3、mysql.md) | yum |
| 4 | Nginx | [4、nginx.md](./4、nginx.md) | yum / 编译 |
| 5 | Redis | [5、redis.md](./5、redis.md) | yum / 编译 |
| 6 | Oracle | [6、oracle.md](./6、oracle.md) | 二进制 |
| 7 | FRP | [7、frp.md](./7、frp.md) | 二进制 |
| 8 | Zookeeper | [8、zookeeper.md](./8、zookeeper.md) | 仅 ZK 模式 Kafka；新集群用 KRaft 可跳过 |
| 9 | Kafka | [9、kafka.md](./9、kafka.md) | 二进制 KRaft，≥3 节点 |
| 10 | RabbitMQ | [10、RabbitMq.md](./10、RabbitMq.md) | yum |
| 11 | FastDFS | [11、fastdfs.md](./11、fastdfs.md) | 原生部署；新项目可评估 MinIO |
| 12 | Jenkins | [12、jenkins.md](./12、jenkins.md) | Docker |
| 13 | Nacos | [13、nacos.md](./13、nacos.md) | 二进制 + MySQL |
| 14 | Portainer | [14、portainer.md](./14、portainer.md) | Docker（管理工具） |

**文档结构**：各服务通常为「简介 → 安装（docker / yum / 二进制）→ **{服务} 配置与运维**（系统参数、生产配置、初始化、备份、防火墙等）→ 专题（主从、集群等）」。

**配置注释规范**：配置文件中的说明写在被注释项**上方**（单独一行 `#`），与正文已有风格一致。

**Docker 与原生安装是否都要保留？** 建议**都保留**，分工如下：

| 方式 | 适用场景 | 说明 |
| ---- | -------- | ---- |
| **Docker** | 开发 / 测试 / PoC / 小团队快速落地 | 上手快，但核心库（MySQL、Kafka 等）生产高可用需额外规范 |
| **yum / 二进制** | **正式生产**、需调优、主从/集群 | 文档重点，面试和运维也主要考这套 |
| 两者对照 | 同一服务先看「生产推荐」列 | 不必删 Docker，用来本地验证 compose 和配置项 |

Docker compose 与挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)。偏生产示例配置见 [docker/conf/](./docker/conf/) 与 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md)。

## Docker

Docker 部署与 yum/二进制安装**分开存放**：

| 部署方式 | 数据目录 | 说明 |
| -------- | -------- | ---- |
| Docker | `/data/docker/<服务名>/` | 见 [docker/0、目录规划.md](./docker/0、目录规划.md) |
| yum / 二进制 | `/data/<服务名>/<版本>/` | 见下方「编译或解压」 |
| Docker 偏生产 | `docker/conf/` + compose 改项 | 见 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md) |

公共步骤（安装 Docker、镜像加速、离线传镜像）见 [docker/1、环境准备.md](./docker/1、环境准备.md)。

## yum

| 文件类型   | 路径示例                             | 说明                                |
| ---------- | ------------------------------------ | ----------------------------------- |
| 可执行文件 | `/usr/bin/`、`/usr/sbin/`            | 用户/系统执行命令所在路径           |
| 配置文件   | `/etc/软件名/`                       | 主配置文件目录                      |
| 日志文件   | `/var/log/软件名/`                   | 各类日志输出目录                    |
| 数据文件   | `/var/lib/软件名/`                   | 运行过程中产生的长期数据            |
| 缓存文件   | `/var/cache/软件名/`                 | 可以重新生成的缓存数据              |
| 运行时文件 | `/run/软件名/` 或 `/var/run/软件名/` | PID 文件、Socket 文件、临时状态文件 |
| 服务脚本   | `/etc/systemd/system/`               | Systemd 服务文件                    |
| 库文件     | `/usr/lib/` 或 `/usr/lib64/`         | 动态库文件（.so）                   |
| 临时文件   | `/tmp/`、`var/tmp/`                  | 临时使用的数据文件（可随时清理）    |

## 系统参数（生产通用）

部署 MySQL、Kafka、Nginx、Redis 等前，建议先调整以下系统限制（Oracle 等有专用参数，见各服务文档）。

### 文件描述符 limits.conf

连接数、打开文件数不足时，常见报错 `Too many open files`。

```bash
vi /etc/security/limits.conf

# 单用户可打开的最大文件数（句柄）
* soft nofile 65536
* hard nofile 65536
# 单用户最大进程数（Java / Erlang 等服务）
* soft nproc  65536
* hard nproc  65536
```

修改后**重新登录**生效；systemd 服务还可在 `.service` 中加 `LimitNOFILE=65536`。

### 内核参数 sysctl（按需）

```bash
vi /etc/sysctl.conf

# 全连接队列长度，高并发 Nginx / Redis 建议
net.core.somaxconn = 65535
# 允许内存 overcommit，Redis 持久化 fork 时建议开启
vm.overcommit_memory = 1
# 可用端口范围（大量短连接时）
net.ipv4.ip_local_port_range = 1024 65535

sysctl -p
```

### Redis 透明大页

Redis 官方建议关闭 THP，否则 fork 持久化时延迟增大：

```bash
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

### 防火墙

生产环境按各服务文档开放端口，原则：**仅对应用网段放行**，管理端口不对公网。

## 编译或解压

使用软链接来控制多版本，以 redis 为例

```
/opt/redis/
├── 7.2.4/
│   └── bin/
└── current -> 7.2.4

/data/redis/
├── 7.2.4/
│   ├── conf/        ← redis.conf 等
│   ├── logs/        ← redis.log
│   ├── data/        ← dump.rdb / appendonly.aof
│   ├── tmp/         ← 临时文件（如 RDB 重写）
│   └── redis.pid
└── current -> 7.2.4
```

常用目录

| 文件类型   | 推荐路径示例                           | 说明                                |
| ---------- | -------------------------------------- | ----------------------------------- |
| 源码包     | `/opt/src/`                            | 用于存放各个软件源码包              |
| 可执行文件 | `/opt/<软件名>/<版本>/bin/`            | 用户/系统执行命令所在路径           |
| 配置文件   | `/data/<软件名>/<版本>/conf/`          | 主配置文件目录                      |
| 日志文件   | `/data/<软件名>/<版本>/logs/`          | 各类日志输出目录                    |
| 数据文件   | `/data/<软件名>/<版本>/data/`          | 运行过程中产生的长期数据            |
| 缓存文件   | `/data/<软件名>/<版本>/cache/`（如有） | 可以重新生成的缓存数据              |
| 运行时文件 | `/data/<软件名>/<版本>/run/`           | PID 文件、Socket 文件、临时状态文件 |
| 服务脚本   | `/etc/systemd/system/<软件名>.service` | Systemd 服务文件                    |
| 库文件     | `/opt/<软件名>/<版本>/lib/`（如有）    | 动态库文件（.so）                   |
| 临时文件   | `/data/<软件名>/<版本>/tmp/`           | 临时使用的数据文件                  |
| 环境变量   | `/etc/profile.d/<软件名>.sh`           | 配置环境变量                        |

多版本切换

```bash
ln -sfn /opt/<软件名>/<版本> /opt/nginx/current
ln -sfn /data/<软件名>/<版本> /data/nginx/current

# 重启服务即可应用新版本
sudo systemctl restart nginx
```

> -s：创建软连接
>
> -f：强制执行，如果目标链接已存在则先删除它
>
> -n：当目标是一个符号链接时，不解引用它，防止将目录链接解开，避免报错或错误操作

环境变量

```bash
echo 'export PATH=/opt/<软件名>/current/bin:$PATH' >> /etc/profile.d/<软件名>.sh

chmod +x /etc/profile.d/<软件名>.sh

source /etc/profile.d/<软件名>.sh
```



# 集群分发脚本

```bash
vi /usr/bin/xsync
```

```bash
#!/bin/bash

#1. 判断参数个数
if [ $# -lt 1 ]
then
    echo Not Enough Arguement!
    exit;
fi

#2. 遍历集群所有机器
for host in hadoop000 hadoop001 hadoop002
do
    echo ====================  $host  ====================
    #3. 遍历所有目录，挨个发送

    for file in $@
    do
        #4. 判断文件是否存在
        if [ -e $file ]
            then
                #5. 获取父目录
                pdir=$(cd -P $(dirname $file); pwd)

                #6. 获取当前文件的名称
                fname=$(basename $file)
                ssh $host "mkdir -p $pdir"
                rsync -av $pdir/$fname $host:$pdir
            else
                echo $file does not exists!
        fi
    done
done
```

```bash
# 下载 rsync
yum install -y rsync
# 对脚本授权
chmod +755 /usr/bin/xsync
# 进行集群分发
xsync module/kafka/
```

