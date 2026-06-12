# Zookeeper 简介

[Zookeeper 官网](https://zookeeper.apache.org) | [下载](https://zookeeper.apache.org/releases.html)

Zookeeper 是分布式**协调服务**，提供配置管理、命名服务、分布式锁、Leader 选举等能力。

| 功能 | 说明 |
| ---- | ---- |
| 节点注册 | 临时/持久 znode |
| 集群协调 | Kafka（ZK 模式）、Hadoop 等依赖 |
| Watch 机制 | 监听节点变化 |

常用端口：

| 端口 | 说明 |
| ---- | ---- |
| 2181 | 客户端连接 |
| 2888 / 3888 | 集群内部通信（Follower / Leader 选举） |

## 部署方式推荐

| 环境 | 推荐方式 | 说明 |
| ---- | -------- | ---- |
| **新 Kafka 集群** | **不部署** | 使用 [9、kafka.md](./9、kafka.md) KRaft 模式，无需 Zookeeper |
| 生产（ZK 模式 Kafka） | **二进制** | 至少 3 节点集群 |
| 开发 / 测试 | Docker | 单机快速验证 |

> Kafka **KRaft 模式**不需要 Zookeeper。本文档适用于 ZK 模式 Kafka 或独立 ZK 集群。

---

# docker 安装（Zookeeper 3.8）

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)  
> 偏生产说明见 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md)

compose 文件：[docker/zookeeper.yml](./docker/zookeeper.yml)（独立部署，与 Kafka 分离）

## 1、创建目录

```bash
mkdir -p /data/docker/zookeeper/{data,logs}
```

## 2、偏生产配置（环境变量）

bitnami ZK 主要靠环境变量。偏生产建议修改 [docker/zookeeper.yml](./docker/zookeeper.yml)：

- 生产设置 `ALLOW_ANONYMOUS_LOGIN: "no"` 并配置认证（开发可保持 `yes`）
- 三节点 = 三台宿主机各起一份 compose

堆内存、四字命令等见下文 [Zookeeper 配置与运维](#zookeeper-配置与运维)。

## 3、启动

```bash
cd /path/to/Deploy/docker
docker compose -f zookeeper.yml up -d
```

## 4、测试

```bash
docker exec -it zookeeper zkCli.sh
ls /
```

# 二进制包安装（Zookeeper 3.8.4）

## 1、用户和目录创建

```bash
# 创建用户（如果没有）
sudo useradd -r -s /sbin/nologin zookeeper

# 创建必要目录（安装目录、数据、日志）
sudo mkdir -p /opt/zookeeper
sudo mkdir -p /data/zookeeper/data
sudo mkdir -p /data/zookeeper/logs

# 赋权给 zookeeper 用户
sudo chown -R zookeeper:zookeeper /opt/zookeeper
sudo chown -R zookeeper:zookeeper /data/zookeeper
```

## 2、安装

```bash
cd /opt
sudo tar -zxvf apache-zookeeper-3.8.4-bin.tar.gz -C /opt/zookeeper --strip-components=1
sudo chown -R zookeeper:zookeeper /opt/zookeeper
```

## 3、配置文件

修改配置文件

```bash
mv /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg

vi /opt/zookeeper/conf/zoo.cfg

# 集群数据存储目录
dataDir=/data/zookeeper/data

# 事务日志目录
dataLogDir=/data/zookeeper/logs

# 服务器监听端口
clientPort=2181

# 集群成员配置（id 与 myid 文件对应，2888:3888 为集群通信端口）
server.1=192.168.1.101:2888:3888
server.2=192.168.1.102:2888:3888
server.3=192.168.1.103:2888:3888

# 会话超时
tickTime=2000
initLimit=10
syncLimit=5
```

修改 logback

```bash
vi /opt/zookeeper/conf/logback.xml
```

```xml
<configuration>

  <property name="zookeeper.console.threshold" value="INFO" />

  <property name="zookeeper.log.dir" value="/data/zookeeper/logs/" />
  <property name="zookeeper.log.file" value="zookeeper.log" />
  <property name="zookeeper.log.threshold" value="INFO" />
  <property name="zookeeper.log.maxfilesize" value="256MB" />
  <property name="zookeeper.log.maxbackupindex" value="20" />

  <!--
    Add ROLLINGFILE to root logger to get log file output
  -->
  <appender name="ROLLINGFILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <File>${zookeeper.log.dir}/${zookeeper.log.file}</File>
    <encoder>
      <pattern>%d{ISO8601} [myid:%X{myid}] - %-5p [%t:%C{1}@%L] - %m%n</pattern>
    </encoder>
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
      <level>${zookeeper.log.threshold}</level>
    </filter>
    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
      <maxIndex>${zookeeper.log.maxbackupindex}</maxIndex>
      <FileNamePattern>${zookeeper.log.dir}/${zookeeper.log.file}.%i</FileNamePattern>
    </rollingPolicy>
    <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
      <MaxFileSize>${zookeeper.log.maxfilesize}</MaxFileSize>
    </triggeringPolicy>
  </appender>

  <root level="INFO">
    <appender-ref ref="ROLLINGFILE" />
  </root>
</configuration>
```

## 4、环境变量

```bash
vi /etc/profile.d/zookeeper.sh

export ZOOKEEPER_HOME=/opt/zookeeper
export PATH=$PATH:$ZOOKEEPER_HOME/bin
export ZOO_LOG_DIR=/data/zookeeper/logs

source /etc/profile.d/zookeeper.sh
```

## 5、创建服务

```bash
sudo tee /etc/systemd/system/zookeeper.service > /dev/null <<EOF
[Unit]
Description=Apache ZooKeeper Server
After=network.target

[Service]
Type=forking
User=zookeeper
Group=zookeeper
Environment="ZOO_LOG_DIR=/data/zookeeper/logs"
Environment="JAVA_HOME=/opt/jdk/jdk11"
ExecStart=/opt/zookeeper/bin/zkServer.sh start
ExecStop=/opt/zookeeper/bin/zkServer.sh stop
ExecReload=/opt/zookeeper/bin/zkServer.sh restart
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
TimeoutStartSec=30
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
EOF
```

## 6、同步集群文件

```bash
xsync /opt/zookeeper
xsync /data/zookeeper
xsync /etc/profile.d/zookeeper.sh
xsync /etc/systemd/system/zookeeper.service

# 三台机器分别在 dataDir 下创建 myid（每台只执行对应一行）
echo "1" | sudo tee /data/zookeeper/data/myid   # 节点 1
# echo "2" | sudo tee /data/zookeeper/data/myid   # 节点 2
# echo "3" | sudo tee /data/zookeeper/data/myid   # 节点 3

sudo chown -R zookeeper:zookeeper /data/zookeeper
```

## 7、服务启动

```bash
sudo systemctl daemon-reload
sudo systemctl enable zookeeper
sudo systemctl start zookeeper
sudo systemctl stop zookeeper
# 查看状态
sudo systemctl status zookeeper
```

## 8、测试功能

```bash
# 查询节点状态
/opt/zookeeper/bin/zkServer.sh status

# 连接客户端
/opt/zookeeper/bin/zkCli.sh -server 127.0.0.1:2181

create /test_node "hello"
get /test_node
set /test_node "world"
get /test_node
delete /test_node
```

# Zookeeper 配置与运维

## 1、配置文件

```bash
/opt/zookeeper/conf/zoo.cfg

# 数据地址
dataDir=/data/zookeeper/data
# 事务日志目录
dataLogDir=/data/zookeeper/logs
# 运行日志路径见 conf/logback.xml 中的 zookeeper.log.dir，不要写在 zoo.cfg 里
# 限制每个客户端主机最多可以建立的 ZooKeeper 连接数 默认60
maxClientCnxns=100
# ZooKeeper 内部的时间单位，单位是毫秒
tickTime=2000
# follower 连接 leader 时，最多允许多少个 tickTime 的时间来完成初始化（如数据同步）
initLimit=10
# 运行时 leader 和 follower 之间最多允许多少个 tickTime 的时间未同步数据
syncLimit=5
# 集群配置
server.1=hadoop001:2888:3888
server.2=hadoop002:2888:3888
server.3=hadoop003:2888:3888


# 每小时自动清理一次旧的日志/快照文件
autopurge.purgeInterval=1
# 保留最近 3 个 snapshot 文件（对应部分 log 文件也会保留）
autopurge.snapRetainCount=3
```

## 2、堆内存修改

ZooKeeper 存的是元数据，**堆内存一般固定档位即可**，不必随物理内存线性增长。

| 物理内存 | 建议 `-Xmx` | 说明 |
| -------- | ----------- | ---- |
| 8GB 及以下 | **1GB～2GB** | 小规模或开发 |
| 8GB～32GB | **2GB**（三节点集群常用） | 元数据量正常时够用 |
| 32GB 以上或 znode 极多 | **2GB～4GB** | 一般不超过 **4GB** |

```bash
vi /opt/zookeeper/bin/zkEnv.sh

# 三节点集群常用 2G；元数据量大时可改为 4G
export JVMFLAGS="-Xms2g -Xmx2g -XX:+UseG1GC"
```

## 3、生产环境 JVM 与四字命令

```bash
# zoo.cfg 建议开启（见上文 autopurge 配置）
# 四字命令仅本地可用，避免暴露到公网
4lw.commands.whitelist=stat, ruok, conf, isro
```

## 4、系统参数

Java 进程需调大文件描述符，通用模板见 [0、readme.md](./0、readme.md#系统参数生产通用)。

```bash
# /etc/security/limits.conf
zookeeper soft nofile 65536
zookeeper hard nofile 65536

# systemd 服务中可加 LimitNOFILE=65536
```

## 5、集群状态检查

```bash
/opt/zookeeper/bin/zkServer.sh status
echo ruok | nc localhost 2181    # 应返回 imok
echo stat | nc localhost 2181
```

## 6、防火墙

```bash
sudo firewall-cmd --permanent --add-port=2181/tcp
sudo firewall-cmd --permanent --add-port=2888/tcp
sudo firewall-cmd --permanent --add-port=3888/tcp
sudo firewall-cmd --reload
```

> 2181 仅对 Kafka/客户端网段开放；2888/3888 仅集群节点互通。

## 7、备份

```bash
# 定期备份 dataDir 和 dataLogDir（ZK 停止或使用 snapshot 机制）
tar -czf /backup/zookeeper_$(date +%F).tar.gz /data/zookeeper/
```

## 8、监控要点

- 磁盘空间：`dataLogDir` 写满会导致 ZK 不可用
- 会话数：`echo stat | nc localhost 2181` 查看 connections
- 延迟：`mntr` 命令查看 avg/max latency（需加入 whitelist）
