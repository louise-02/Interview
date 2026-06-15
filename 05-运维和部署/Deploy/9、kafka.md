# Kafka 简介

[Kafka官网](https://kafka.apache.org/downloads) 中选择 Binary download 获取包。

## 部署模式说明

| 模式 | Zookeeper | 文档章节 | 说明 |
| ---- | --------- | -------- | ---- |
| **KRaft** | 不需要 | Docker KRaft / 二进制 KRaft | Kafka 3.3+ 内置 Raft 管理元数据，**新部署推荐** |
| ZK 模式 | 需要 | Docker+ZK / 二进制 ZK | 传统架构，老集群维护 |

> 3.0 版本以后建议使用 JDK 11；2.8 及以前使用 JDK 8。KRaft 与 ZK 模式**不能共用同一数据目录**，切换模式需清空 `/data/kafka/data`。

## 部署方式推荐

| 环境 | 推荐方式 | 说明 |
| ---- | -------- | ---- |
| **生产（新集群）** | **二进制 KRaft，≥3 节点** | 官方主流方向，无需 Zookeeper |
| 生产（容器化团队） | Docker KRaft，≥3 节点 | 团队已统一 Docker 时可选 |
| 开发 / 测试 | Docker KRaft 单节点 | 快速搭建 |
| 老集群维护 | 二进制或 Docker + ZK | 不再建议新项目使用 |

> **安装方式**：Docker 章节用于本地验证与 compose 练习；**正式生产**请看「二进制安装」+ 下文「Kafka 配置与运维」。两者都保留，不必删 Docker。

常用端口：

| 端口 | 说明 |
| ---- | ---- |
| 9092 | Broker 客户端连接 |
| 9093 | KRaft Controller 内部通信 |

---

# docker 安装（Kafka 3.8 KRaft，推荐）

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)  
> 偏生产说明见 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md)

不需要单独部署 Zookeeper。compose 文件：[docker/kafka-kraft.yml](./docker/kafka-kraft.yml)

> Docker 生产环境建议 **≥3 节点** 分别部署；单节点 compose 仅适合开发测试。`advertised.listeners` 必须填客户端可达的真实 IP/域名。

## 1、创建目录

```bash
mkdir -p /data/docker/kafka/data
```

## 2、偏生产配置（环境变量）

Kafka（bitnami）主要靠 **compose 环境变量**，一般不必挂 `server.properties`。偏生产可参考 [docker/kafka-kraft.prod.example.yml](./docker/kafka-kraft.prod.example.yml)：

- 必改 `KAFKA_CFG_ADVERTISED_LISTENERS` 为本机 IP
- 建议 `KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=false`
- 三节点时每台各一份 compose，分别改 `NODE_ID`、`QUORUM_VOTERS`、副本相关 env

副本、`min.insync.replicas` 说明见下文 [Kafka 配置与运维](#kafka-配置与运维)。

## 3、启动

```bash
cd /path/to/Deploy/docker
# 将 kafka-kraft.yml 中 KAFKA_CFG_ADVERTISED_LISTENERS 的「宿主机IP」改为实际 IP
docker compose -f kafka-kraft.yml up -d
```

## 4、测试

```bash
docker exec kafka kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test --partitions 1 --replication-factor 1
docker exec kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
```

Kafka UI 为可选组件，见下文 [Kafka UI](#kafka-uidocker-latest可选)。

---

# docker 安装（Kafka 3.8 + ZK 3.8）

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)  
> 偏生产说明见 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md)

**需先部署 Zookeeper**，见 [8、zookeeper.md](./8、zookeeper.md)。Kafka 与 Zookeeper 使用独立 compose 文件，可单独部署。

compose 文件：[docker/kafka-zk.yml](./docker/kafka-zk.yml)

## 1、创建目录

```bash
mkdir -p /data/docker/kafka/data
```

## 2、偏生产配置（环境变量）

修改 [kafka-zk.yml](./docker/kafka-zk.yml) 中 `KAFKA_CFG_ZOOKEEPER_CONNECT`、`KAFKA_CFG_ADVERTISED_LISTENERS` 为真实 IP；建议关闭自动建 topic。ZK 偏生产见 [8、zookeeper.md](./8、zookeeper.md) Docker 章节。

## 3、启动

```bash
cd /path/to/Deploy/docker
# 修改 kafka-zk.yml 中 KAFKA_CFG_ZOOKEEPER_CONNECT、KAFKA_CFG_ADVERTISED_LISTENERS 的「宿主机IP」
docker compose -f kafka-zk.yml up -d
```

## 4、测试

```bash
docker exec kafka kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test --partitions 1 --replication-factor 1
docker exec kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
```

---

# Kafka UI（Docker latest，可选）

Web 管理界面，按需部署。compose 文件：[docker/kafka-ui.yml](./docker/kafka-ui.yml)

## 1、与 Kafka 一起使用

先启动 Kafka（KRaft 或 ZK 模式），再部署 UI：

```bash
cd /path/to/Deploy/docker
# 修改 BOOTSTRAPSERVERS 为宿主机 IP；ZK 模式还需取消 ZOOKEEPER 环境变量注释
docker compose -f kafka-ui.yml up -d
```

访问 `http://宿主机IP:8080`

## 2、单独连接已有集群

```bash
# KRaft 集群（无需 ZOOKEEPER 环境变量）
docker run -d \
  --name kafka-ui \
  -p 8080:8080 \
  -e KAFKA_CLUSTERS_0_NAME=dev-cluster \
  -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=10.168.106.107:9092,10.168.106.108:9092,10.168.106.109:9092 \
  provectuslabs/kafka-ui:latest

# ZK 模式集群（可选填 ZOOKEEPER 地址）
docker run -d \
  --name kafka-ui \
  -p 8080:8080 \
  -e KAFKA_CLUSTERS_0_NAME=dev-cluster \
  -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=10.168.106.107:9092,10.168.106.108:9092,10.168.106.109:9092 \
  -e KAFKA_CLUSTERS_0_ZOOKEEPER=10.168.106.107:2181,10.168.106.108:2181,10.168.106.109:2181 \
  provectuslabs/kafka-ui:latest
```

# 二进制包安装（Kafka 3.8.1 + ZK）

## 1、用户和目录创建

```bash
# 创建 kafka 用户（如果没有）
sudo useradd -r -s /sbin/nologin kafka

# 创建必要目录（安装目录、数据、日志）
sudo mkdir -p /opt/kafka
sudo mkdir -p /data/kafka/data
sudo mkdir -p /data/kafka/logs

# 赋权给 kafka 用户
sudo chown -R kafka:kafka /opt/kafka
sudo chown -R kafka:kafka /data/kafka
```

## 2、安装 Kafka

```bash
cd /opt
# 假设 kafka 安装包已上传
sudo tar -zxvf kafka_2.13-3.8.1.tgz -C /opt/kafka --strip-components=1
sudo chown -R kafka:kafka /opt/kafka
```

## 3、配置文件修改

```bash
/opt/kafka/config/server.properties

broker.id=1
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://ip:9092
log.dirs=/data/kafka/data
zookeeper.connect=192.168.1.101:2181,192.168.1.102:2181,192.168.1.103:2181/kafka
# ... 其余生产参数见下文 ZK 模式生产配置
```

> broker.id 依次为 1 2 3

## 4、环境变量配置

```bash
vi /etc/profile.d/kafka.sh

export KAFKA_HOME=/opt/kafka
export PATH=$PATH:$KAFKA_HOME/bin

source /etc/profile.d/kafka.sh
```

## 5、创建服务

```bash
sudo tee  /etc/systemd/system/kafka.service > /dev/null <<EOF
[Unit]
Description=Apache Kafka Server
After=network.target zookeeper.service

[Service]
User=kafka
Group=kafka
Environment="JAVA_HOME=/opt/jdk/jdk11"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
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
xsync /opt/kafka
xsync /data/kafka
xsync /etc/profile.d/kafka.sh
xsync /etc/systemd/system/kafka.service

# 修改每台机器的配置
vi /opt/kafka/config/server.properties

# 依次改为 1 2 3
broker.id=1
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://ip:9092
```

## 7、服务启动

```bash
sudo systemctl daemon-reload
sudo systemctl enable kafka
sudo systemctl start kafka
sudo systemctl status kafka
```

非服务启动

```bash
# 启动 kafka
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties

# 关闭 kafka
/opt/kafka/bin/kafka-server-stop.sh
```

> **停集群顺序**：先停完所有 Kafka 节点，再停 Zookeeper。若 ZK 先停，Kafka 无法优雅下线，只能手动 kill 进程。

## 8、常见问题

```
// 问题1：Cluster ID 不匹配
The Cluster ID xxx doesn't match stored clusterId
// 解决：删除 server.properties 中 log.dirs 目录下的所有文件后重启

// 问题2：Zookeeper 选举失败
Cannot open channel to 2 at election address
// 解决：检查防火墙是否关闭；检查各节点 /etc/hosts 是否一致
```

# 二进制包安装（Kafka 3.8.1 KRaft）

与 ZK 模式共用安装目录 `/opt/kafka` 和数据目录 `/data/kafka/data`，但**不能**在已有 ZK 模式数据的目录上直接切换，需重新格式化。

## 1、用户和目录创建

同上文「二进制包安装（Kafka 3.8.1 + ZK）」第 1 步。

## 2、安装 Kafka

```bash
cd /opt
sudo tar -zxvf kafka_2.13-3.8.1.tgz -C /opt/kafka --strip-components=1
sudo chown -R kafka:kafka /opt/kafka
```

## 3、生成 Cluster ID 并格式化存储

```bash
# 生成 cluster id（集群内所有节点使用同一个 id）
/opt/kafka/bin/kafka-storage.sh random-uuid
# 输出示例：MkU3OEVBNTcwNTJENDM2Qk

# 首次部署时，每台机器各执行一次 format（CLUSTER_ID 三台相同）
sudo -u kafka /opt/kafka/bin/kafka-storage.sh format \
  -t MkU3OEVBNTcwNTJENDM2Qk \
  -c /opt/kafka/config/kraft/server.properties
```

> `format` 会清空 `log.dirs` 指定目录下的数据，仅在**首次部署**或**重建集群**时执行。

## 4、配置文件修改

以三台节点为例，每台修改 `/opt/kafka/config/kraft/server.properties`（**完整生产项见下文 [KRaft 模式生产配置](#3kraft-模式生产配置)**）：

每台仅改 **`node.id`** 和 **`advertised.listeners`**（其余三台相同）：

| 节点 | node.id | advertised.listeners |
| ---- | ------- | -------------------- |
| 192.168.1.101 | 1 | PLAINTEXT://192.168.1.101:9092 |
| 192.168.1.102 | 2 | PLAINTEXT://192.168.1.102:9092 |
| 192.168.1.103 | 3 | PLAINTEXT://192.168.1.103:9092 |

> KRaft 模式**不需要** `zookeeper.connect` 和 `broker.id`，使用 `node.id` 代替。

## 5、环境变量配置

同 ZK 模式，见上文第 4 步。

## 6、创建服务

```bash
sudo tee /etc/systemd/system/kafka.service > /dev/null <<EOF
[Unit]
Description=Apache Kafka Server (KRaft)
After=network.target

[Service]
User=kafka
Group=kafka
Environment="JAVA_HOME=/opt/jdk/jdk11"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
TimeoutStartSec=30
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
EOF
```

## 7、同步集群文件

```bash
xsync /opt/kafka
xsync /data/kafka
xsync /etc/profile.d/kafka.sh
xsync /etc/systemd/system/kafka.service

# 每台机器修改 node.id 和 advertised.listeners
vi /opt/kafka/config/kraft/server.properties
```

## 8、服务启动

```bash
sudo systemctl daemon-reload
sudo systemctl enable kafka
sudo systemctl start kafka
sudo systemctl status kafka
```

非服务启动：

```bash
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/kraft/server.properties
/opt/kafka/bin/kafka-server-stop.sh
```

# Kafka 配置与运维

> KRaft 与 ZK 模式**配置差异较大**（不同配置文件、不同核心参数、KRaft 需 format），但**安装包相同**。分章节写在本文档即可，**不必单独开文件**；老 ZK 模式内容保留供维护旧集群参考。

## ZK 模式 vs KRaft 配置差异

| 项目 | ZK 模式 | KRaft 模式（3.3+，推荐） |
| ---- | ------- | ------------------------ |
| 配置文件 | `config/server.properties` | `config/kraft/server.properties` |
| 节点标识 | `broker.id` | `node.id` |
| 元数据 | `zookeeper.connect` | `process.roles` + `controller.quorum.voters` |
| 监听端口 | 9092 | 9092（Broker）+ **9093**（Controller） |
| 首次部署 | 直接启动 | `kafka-storage.sh format` 格式化目录 |
| 依赖 | 需 [8、zookeeper.md](./8、zookeeper.md) | 无需 Zookeeper |

## 1、日志与堆内存（两种模式通用）

Kafka **堆内存不宜按物理内存比例无脑拉大**：Broker 读写日志主要依赖 **OS Page Cache**，堆过大反而挤占页缓存。

| 部署方式 | JVM 堆（`-Xmx` / `-Xms`） | 说明 |
| -------- | ------------------------- | ---- |
| **专用 Broker 节点** | **4GB～8GB** 常见 | 32GB 机器仍建议约 **8GB**（约 25%），余量留给 Page Cache |
| 64GB 及以上 | **8GB～12GB** | 一般不超过 **12GB**，再大收益递减 |
| 与 ZK / 其他混部 | **4GB～6GB** | 按剩余内存酌减 |

示例：32GB 专用节点 → `-Xmx8G -Xms8G`；16GB 专用 → `-Xmx6G -Xms6G`。

```bash
# 运行日志目录
vi /opt/kafka/bin/kafka-run-class.sh
# 将 LOG_DIR 改为 /data/kafka/logs

# JVM 堆内存（Xms 与 Xmx 设相同，避免运行时扩容）
vi /opt/kafka/bin/kafka-server-start.sh
# export KAFKA_HEAP_OPTS="-Xmx8G -Xms8G"
```

## 2、ZK 模式生产配置

文件：`/opt/kafka/config/server.properties`（三节点示例，与 [Kafka 简介](../../02-技术栈/Kafka/1、Kafka%20简介.md) 对齐；仅维护旧集群时使用）

```properties
# ========== 节点标识（每台不同：1 / 2 / 3）==========
# broker 的全局唯一编号，不能重复，只能是数字
broker.id=1

# ========== 线程与网络 buffer ==========
# 处理网络请求的线程数量
num.network.threads=3
# 用来处理磁盘 IO 的线程数量
num.io.threads=8
# 发送套接字的缓冲区大小
socket.send.buffer.bytes=102400
# 接收套接字的缓冲区大小
socket.receive.buffer.bytes=102400
# 请求套接字的最大字节数（须大于 message.max.bytes）
socket.request.max.bytes=104857600

# ========== 数据目录 ==========
# kafka 消息数据目录（与安装目录分离，建议挂载独立磁盘）
log.dirs=/data/kafka/data
# 用来恢复和清理 log.dirs 下数据的线程数量
num.recovery.threads.per.data.dir=1

# ========== 三节点集群：副本与分区默认值 ==========
# topic 手动创建时的默认分区数参考（按业务调整）
num.partitions=3
# 业务 topic 默认副本数（三节点设为 3，须 ≤ broker 数）
default.replication.factor=3
# 内部 offset 主题 __consumer_offsets 的副本数
offsets.topic.replication.factor=3
# Kafka 事务内部 topic 的副本数
transaction.state.log.replication.factor=3
# 事务内部 topic 写入时至少几个 ISR 副本确认
transaction.state.log.min.isr=2
# Producer 设 acks=all 时，至少几个 ISR 副本写入才算成功（RF=3 时常用 2）
min.insync.replicas=2
# 是否允许非 ISR 副本在 Leader 故障后当选 Leader
unclean.leader.election.enable=false
# Broker 接受的单条消息最大字节数；Producer max.request.size 须 ≤ 此值
message.max.bytes=10485760
# Follower 从 Leader 拉取时的单次最大字节数；建议 ≥ message.max.bytes
replica.fetch.max.bytes=10485760

# ========== 日志保留 ==========
# segment 文件保留的最长时间（168=7 天）
log.retention.hours=168
# 每个 segment 文件的大小，默认最大 1G
log.segment.bytes=1073741824
# 检查过期数据的时间间隔，默认 5 分钟检查一次
log.retention.check.interval.ms=300000
# 禁止访问不存在的 topic 时自动创建（须手动建 topic）
auto.create.topics.enable=false
# 允许删除 topic（配合 kafka-topics.sh --delete）
delete.topic.enable=true

# ========== 网络监听（每台 advertised 填本机 IP）==========
# Broker 在本机绑定的监听地址
listeners=PLAINTEXT://0.0.0.0:9092
# 告诉客户端应连接的地址；每台填本机 IP
advertised.listeners=PLAINTEXT://192.168.1.101:9092

# ========== Zookeeper（三节点，ZK 模式专有）==========
# 连接 Zookeeper 集群地址（chroot /kafka 方便管理）
zookeeper.connect=192.168.1.101:2181,192.168.1.102:2181,192.168.1.103:2181/kafka
# 连接 ZK 超时（ms）
zookeeper.connection.timeout.ms=18000
```

## 3、KRaft 模式生产配置

文件：`/opt/kafka/config/kraft/server.properties`（三节点示例，**新集群推荐**；Broker 侧参数与 ZK 模式保持一致）

```properties
# ========== KRaft 角色（合并 broker+controller，小中集群常用）==========
# 本节点承担的角色：broker=处理消息读写；controller=管理元数据与选举；可写 broker,controller
process.roles=broker,controller
# 节点全局唯一编号（替代 ZK 模式的 broker.id）；三台分别为 1 / 2 / 3
node.id=1
# Controller 仲裁成员列表，格式 nodeId@host:controllerPort；三台都填相同内容
controller.quorum.voters=1@192.168.1.101:9093,2@192.168.1.102:9093,3@192.168.1.103:9093

# ========== 线程与网络 buffer ==========
# 处理网络请求的线程数量
num.network.threads=3
# 用来处理磁盘 IO 的线程数量
num.io.threads=8
# 发送套接字的缓冲区大小
socket.send.buffer.bytes=102400
# 接收套接字的缓冲区大小
socket.receive.buffer.bytes=102400
# 请求套接字的最大字节数（须大于 message.max.bytes）
socket.request.max.bytes=104857600

# ========== 监听（KRaft 比 ZK 多 9093 Controller 端口）==========
# PLAINTEXT=客户端/Broker 通信；CONTROLLER=Controller 节点间元数据通信
listeners=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
# 告诉客户端应连接的地址；每台填本机 IP（Controller 端口一般不对外暴露）
advertised.listeners=PLAINTEXT://192.168.1.101:9092
# 哪个 listener 名称用于 Controller 通信
controller.listener.names=CONTROLLER
# Broker 之间副本同步使用的 listener 名称
inter.broker.listener.name=PLAINTEXT
# listener 名称与安全协议的映射关系
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

# ========== 数据目录 ==========
# kafka 消息数据目录（与 ZK 模式路径相同，但不能混用同一目录数据）
log.dirs=/data/kafka/data
# 用来恢复和清理 log.dirs 下数据的线程数量
num.recovery.threads.per.data.dir=1

# ========== 三节点集群：副本与分区默认值 ==========
# topic 手动创建时的默认分区数参考（按业务调整）
num.partitions=3
# 业务 topic 默认副本数（三节点设为 3，须 ≤ broker 数）
default.replication.factor=3
# 内部 offset 主题 __consumer_offsets 的副本数
offsets.topic.replication.factor=3
# Kafka 事务内部 topic 的副本数
transaction.state.log.replication.factor=3
# 事务内部 topic 写入时至少几个 ISR 副本确认
transaction.state.log.min.isr=2
# Producer 设 acks=all 时，至少几个 ISR 副本写入才算成功（RF=3 时常用 2）
min.insync.replicas=2
# 是否允许非 ISR 副本在 Leader 故障后当选 Leader
unclean.leader.election.enable=false
# Broker 接受的单条消息最大字节数；Producer max.request.size 须 ≤ 此值
message.max.bytes=10485760
# Follower 从 Leader 拉取时的单次最大字节数；建议 ≥ message.max.bytes
replica.fetch.max.bytes=10485760

# ========== 日志保留 ==========
# segment 文件保留的最长时间（168=7 天）
log.retention.hours=168
# 日志总大小上限；-1 表示不限制，仅按 log.retention.hours 删除
log.retention.bytes=-1
# 每个 segment 文件的大小，默认最大 1G
log.segment.bytes=1073741824
# 检查过期数据的时间间隔，默认 5 分钟检查一次
log.retention.check.interval.ms=300000
# 禁止访问不存在的 topic 时自动创建（须手动建 topic）
auto.create.topics.enable=false
# 允许删除 topic（配合 kafka-topics.sh --delete）
delete.topic.enable=true

# 注意：KRaft 模式无 zookeeper.connect、无 broker.id
```

首次部署必须 format（集群共用一个 cluster id）：

```bash
/opt/kafka/bin/kafka-storage.sh random-uuid
sudo -u kafka /opt/kafka/bin/kafka-storage.sh format \
  -t <CLUSTER_ID> -c /opt/kafka/config/kraft/server.properties
```

## 4、系统参数

Kafka 连接数、分区数较多时需调大文件描述符，通用模板见 [0、readme.md](./0、readme.md#系统参数生产通用)。

```bash
# /etc/security/limits.conf
kafka soft nofile 65536
kafka hard nofile 65536

# systemd 的 kafka.service 中
LimitNOFILE=65536
```

## 5、集群验证

```bash
/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server 127.0.0.1:9092
# KRaft 可看元数据
/opt/kafka/bin/kafka-metadata.sh --snapshot /data/kafka/data/__cluster_metadata-0/*.log --print
```

## 6、Topic 管理

```bash
/opt/kafka/bin/kafka-topics.sh --bootstrap-server 127.0.0.1:9092 \
  --create --topic your_topic --partitions 6 --replication-factor 3

/opt/kafka/bin/kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --describe --topic your_topic
```

> 副本数 ≤ broker 数；`min.insync.replicas=2` 时生产端建议 `acks=all`。

## 7、防火墙

```bash
sudo firewall-cmd --permanent --add-port=9092/tcp
sudo firewall-cmd --permanent --add-port=9093/tcp   # 仅 KRaft controller
sudo firewall-cmd --reload
```

## 8、监控与备份

```bash
# 消费 lag
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --describe --all-groups
df -h /data/kafka/data
```

- 配置备份：`/opt/kafka/config/`
- 数据备份：`log.dirs` 目录，建议磁盘快照或按官方离线备份流程
- ZK 模式停集群：**先停 Kafka，再停 Zookeeper**

# Kafka 测试

## 1、topic

查看操作主题命令行参数

```bash
kafka-topics.sh

--bootstrap-server <String: server toconnect to> 连接的 Kafka Broker 主机名称和端口号。
--topic <String: topic> 操作的 topic 名称。
--create 创建主题。
--delete 删除主题。
--alter 修改主题。
--list 查看所有主题。
--describe 查看主题详细描述。
--partitions <Integer: # of partitions> 设置分区数。
--replication-factor<Integer: replication factor> 设置分区副本。
--config <String: name=value> 更新系统默认的配置。
```

常用命令

```bash
# 查看当前服务器中的所有 topic
kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --list

# 创建 first topic，1分区3副本
kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --create --partitions 1 --replication-factor 3 --topic first

# 查看 first 主题的详情

kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --describe --topic first

# 修改分区数（注意：分区数只能增加，不能减少）

kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --alter --topic first --partitions 3

# 删除 topic
kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --delete --topic first
```

## 2、producer

```bash
# 发送消息
kafka-console-producer.sh --bootstrap-server 127.0.0.1:9092  --topic first
```

## 3、consumer

查看操作消费者命令参数

```bash
kafka-console-consumer.sh

--bootstrap-server <String: server toconnect to> 连接的 Kafka Broker 主机名称和端口号。
--topic <String: topic> 操作的 topic 名称。
--from-beginning 从头开始消费。
--group <String: consumer group id> 指定消费者组名称。
```

常用命令

```bash
# 消费 first 主题中的数据
kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic first
# 把主题中所有的数据都读取出来，包括历史数据
kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --from-beginning --topic first
```

## 4、consumer groups

查看消费者组命令命令

```bash
kafka-consumer-groups.sh

--bootstrap-server <String: server toconnect to> 连接的 Kafka Broker 主机名称和端口号。
--describe 查看主题详细描述。
--group <String: consumer group id> 指定消费者组名称。
```

常用命令

```bash
# 查看所有的 group
./kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --list

# 查看 my-group 消费者组的偏移量
./kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --group my-group --describe
```

