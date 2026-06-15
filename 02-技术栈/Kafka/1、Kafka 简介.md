# 1、Kafka 概述

## 1、简介

https://kafka.apache.org/

**Kafka传统定义**：Kafka是一个**分布式**的**基于发布/订阅模式的消息队列**（MessageQueue），主要应用于大数据实时处理领域。

**发布/订阅**：消息的发布者**不会将消息直接发送给特定的订阅者**，而是将发布的消息**分为不同的类别**，**订阅者只接收感兴趣的消息**。

**Kafka最新定义** ： Kafka是 一个开源的**分布式事件流平台**（Event StreamingPlatform），被数千家公司用于高性能**数据管道、流分析、数据集成和关键任务应用**。

![image-20260313090456980](./pictures/image-20260313090456980-3363898.png)

## 2、传统消息队列的应用场景

缓存/消峰、解耦和异步通信。

**缓冲/消峰**：有助于控制和优化数据流经过系统的速度，解决生产消息和消费消息的处理速度不一致的情况。

**解耦**：允许你独立的扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束。

**异步通信**：允许用户把一个消息放入队列，但并不立即处理它，然后在需要的时候再去处理它们。

## 3、消息队列的两种模式

**点对点模式**

消费者主动拉取数据，消息收到后清除消息



**发布/订阅模式**

可以有多个topic主题（浏览、点赞、收藏、评论等）

消费者消费数据之后，不删除数据

每个消费者相互独立，都可以消费到数据

## 4、基本架构

![image-20260313090653040](./pictures/image-20260313090653040-3364014.png)

**Producer**：消息生产者，就是向 Kafka broker 发消息的客户端。

**Consumer**：消息消费者，向 Kafka broker 取消息的客户端。

**Consumer Group（CG）**：消费者组，由多个 consumer 组成。**消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响**。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。

**Broker**：一台 Kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个broker 可以容纳多个 topic。

**Topic**：可以理解为一个队列，**生产者和消费者面向的都是一个topic**。

**Partition**：分区为了实现扩展性，一个非常大的 topic 可以分布到多个 broker（即服务器）上，**一个 topic 可以分为多个 partition**，每个 partition 是一个**有序的队列**。

**Replica**：副本。一个 topic 的每个分区都有若干个副本，一个 Leader 和若干个Follower。

**Leader**：每个分区多个**副本的“主”**，生产者发送数据的对象，以及消费者消费数据的对象都是 Leader。

**Follower**：每个分区多个**副本中的“从”**，实时从 Leader 中同步数据，保持和Leader 数据的同步。Leader 发生故障时，某个 Follower 会成为新的 Leader。





# 2、Kafka 快速入门

## 1、集群部署

目录规范与 [Deploy/9、kafka.md](../../05-运维和部署/Deploy/9、kafka.md)、[Deploy/8、zookeeper.md](../../05-运维和部署/Deploy/8、zookeeper.md) 一致。

```bash
# 规划为三台机器 hadoop000 hadoop001 hadoop002

# ========== 创建目录（安装目录 + 数据目录 + 日志目录）==========
sudo mkdir -p /opt/kafka
sudo mkdir -p /data/kafka/data
sudo mkdir -p /data/kafka/logs
sudo mkdir -p /opt/zookeeper
sudo mkdir -p /data/zookeeper/data
sudo mkdir -p /data/zookeeper/logs

# ========== 安装 Kafka ==========
sudo tar -zxvf kafka_2.13-3.8.1.tgz -C /opt/kafka --strip-components=1

# ========== 安装 Zookeeper（独立部署，与 Kafka 分开）==========
sudo tar -zxvf apache-zookeeper-3.8.4-bin.tar.gz -C /opt/zookeeper --strip-components=1
mv /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg
```

## 2、Kafka 配置文件（三节点）

三台机器：**hadoop000**、**hadoop001**、**hadoop002**（每台 `broker.id` 不同，`advertised.listeners` 填本机 IP）。

**hadoop000 示例**（001、002 仅改 `broker.id` 和 `advertised.listeners`）：

```bash
vim /opt/kafka/config/server.properties

# ========== 节点标识（每台不同：0 / 1 / 2）==========
# broker 的全局唯一编号，不能重复，只能是数字；三台分别为 0 / 1 / 2
broker.id=0

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
# topic 自动创建时的默认分区数（三节点生产常见 3 或 6，按业务调整）
num.partitions=3
# 业务 topic 自动创建时的默认副本数（三节点集群设为 3，须 ≤ broker 数）
default.replication.factor=3
# 内部 offset 主题 __consumer_offsets 的副本数（建议与 default.replication.factor 一致）
offsets.topic.replication.factor=3
# Kafka 事务内部 topic 的副本数
transaction.state.log.replication.factor=3
# 事务内部 topic 写入时至少几个 ISR 副本确认
transaction.state.log.min.isr=2
# Producer 设 acks=all 时，至少几个 ISR 副本写入才算成功（RF=3 时常用 2，允许挂 1 台）
min.insync.replicas=2
# 是否允许非 ISR 副本在 Leader 故障后当选 Leader；false=宁可停写也不丢已确认数据
unclean.leader.election.enable=false
# Broker 接受的单条消息最大字节数；Producer max.request.size 须 ≤ 此值
message.max.bytes=10485760
# Follower 从 Leader 拉取时的单次最大字节数；建议 ≥ message.max.bytes
replica.fetch.max.bytes=10485760

# ========== 日志保留 ==========
# segment 文件保留的最长时间，超时将被删除（168=7 天）
log.retention.hours=168
# 每个 segment 文件的大小，默认最大 1G
log.segment.bytes=1073741824
# 检查过期数据的时间间隔，默认 5 分钟检查一次
log.retention.check.interval.ms=300000
# 禁止 Producer/Consumer 访问不存在的 topic 时自动创建（生产须手动建 topic）
auto.create.topics.enable=false

# ========== 网络监听（每台 advertised 填本机 IP）==========
# Broker 在本机绑定的监听地址；0.0.0.0 表示所有网卡
listeners=PLAINTEXT://0.0.0.0:9092
# 告诉客户端应连接的地址；每台填本机 hostname/IP（hadoop001/002 改成对应主机名）
advertised.listeners=PLAINTEXT://hadoop000:9092

# ========== Zookeeper（三节点）==========
# 连接 Zookeeper 集群地址（在 zk 根目录下创建 /kafka 方便管理）
zookeeper.connect=hadoop000:2181,hadoop001:2181,hadoop002:2181/kafka
```

## 3、Zookeeper 配置文件（三节点）

三台 **`/opt/zookeeper/conf/zoo.cfg` 内容相同**，仅各节点 `dataDir/myid` 不同。

```bash
vim /opt/zookeeper/conf/zoo.cfg

# ========== 基础配置 ==========
# 集群数据存储目录
dataDir=/data/zookeeper/data
# 事务日志目录（与 dataDir 分离，减轻 IO 竞争）
dataLogDir=/data/zookeeper/logs
# 客户端连接端口
clientPort=2181
# 单个客户端最大连接数
maxClientCnxns=100

# ========== 集群超时 ==========
# 心跳基本时间单位（ms），默认 2000；initLimit/syncLimit 均以此为倍数
tickTime=2000
# Follower 初次连接 Leader 的最大心跳次数（tickTime × initLimit = 最长等待时间）
initLimit=10
# Leader 与 Follower 之间同步的最大心跳次数
syncLimit=5

# ========== 集群节点列表 ==========
# server.id=hostname:Leader选举端口:集群通信端口
server.1=hadoop000:2888:3888
server.2=hadoop001:2888:3888
server.3=hadoop002:2888:3888
```

**myid 文件**（在各自 `dataDir` 下创建，内容与 server.N 的 N 一致）：

```bash
# hadoop000
echo 1 > /data/zookeeper/data/myid
# hadoop001
echo 2 > /data/zookeeper/data/myid
# hadoop002
echo 3 > /data/zookeeper/data/myid
```

## 4、集群分发脚本

同步 Deploy 目录到三台机器：

```bash
xsync /opt/kafka
xsync /data/kafka
xsync /opt/zookeeper
xsync /data/zookeeper
```

`xsync` 脚本示例：

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

## 5、环境变量与 JVM 堆内存

**推荐改法：直接改启动脚本**（与 Deploy 文档一致）

```bash
# ========== Kafka 运行日志目录 ==========
vi /opt/kafka/bin/kafka-run-class.sh
# 将 LOG_DIR 改为 /data/kafka/logs

# ========== Kafka Broker 堆内存 ==========
vi /opt/kafka/bin/kafka-server-start.sh
# 找到 KAFKA_HEAP_OPTS 一行，取消注释并改为（Xms=Xmx，避免运行时扩缩容）：
# export KAFKA_HEAP_OPTS="-Xmx6G -Xms6G"

# ========== Zookeeper 堆内存 ==========
vi /opt/zookeeper/bin/zkEnv.sh
export JVMFLAGS="-Xms2g -Xmx2g -XX:+UseG1GC"
```

**环境变量**（可选，方便命令行操作）：

```bash
vi /etc/profile.d/kafka.sh

export KAFKA_HOME=/opt/kafka
export PATH=$PATH:$KAFKA_HOME/bin

source /etc/profile.d/kafka.sh
```

**堆大小参考**

| 组件 | 物理内存 / 场景 | 建议 `-Xmx` | 说明 |
|------|-----------------|-------------|------|
| **Kafka Broker** | 16GB 专用节点 | **6G** | `-Xmx6G -Xms6G` |
| **Kafka Broker** | 32GB 专用节点 | **8G** | 约 25%，余量留给 Page Cache |
| **Kafka Broker** | 与 ZK 等同机混部 | **4G～6G** | 按剩余内存酌减 |
| **Zookeeper** | 三节点集群（8～32GB） | **2G** | 元数据量正常时够用 |
| **Zookeeper** | znode 极多 / 32GB+ | **2G～4G** | 一般不超过 4G |

**备选：启动前临时 export**（仅 Kafka；ZK 堆在 `zkEnv.sh` 改好则无需再设）

```bash
export KAFKA_HEAP_OPTS="-Xmx6G -Xms6G"
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
```

**内存规划参考（每台 16GB，ZK + Kafka 同机）**

| 用途 | 建议 | 说明 |
|------|------|------|
| **Kafka Broker 堆** | 6G | 与 Deploy 文档一致；主要跑请求、索引元数据 |
| **Zookeeper 堆** | 2G | 三节点协调元数据，512M 偏紧 |
| **操作系统 Page Cache** | 尽量留 6G+ | 消息在磁盘 + 页缓存，比堆更重要 |

> 堆不是越大越好：Broker 消息主要在 **log.dirs 磁盘 + OS 缓存**。按 GC、lag 监控再微调。

```bash
# 查看 Broker 进程 JVM 参数
ps aux | grep kafka.Kafka
```

## 6、启动集群（三节点）

每台机器顺序：**先 ZK，后 Kafka**；三台都起来后再建 topic。

**启动 Zookeeper**（每台执行）

```bash
/opt/zookeeper/bin/zkServer.sh start
/opt/zookeeper/bin/zkServer.sh status
# 2181、2888、3888 三台互通；其中一台 2888 为 leader
```

**启动 Kafka**（每台执行；堆已在 `kafka-server-start.sh` 改好则无需再 export）

```bash
# 若未改脚本，16GB 机临时设：export KAFKA_HEAP_OPTS="-Xmx6G -Xms6G"
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
```

**关闭集群**

```bash
# 每台先停 Kafka
/opt/kafka/bin/kafka-server-stop.sh
# 三台 Kafka 都停完后，再停 Zookeeper
/opt/zookeeper/bin/zkServer.sh stop
```

必须先停 Kafka 再停 Zookeeper，否则 Kafka 无法优雅下线。

## 7、常见问题

```bash
# 问题1
The Cluster ID VB7m6OM6SwS5oNJXn20XUA doesn't match stored clusterId
# 解决1
删除 /data/kafka/data 目录下的所有文件后重启

# 问题2
Cannot open channel to 2 at election address
# 解决2
检查各节点的防火墙有没有关闭||检查各节点/etc/hosts内容是否一致
```



# 3、Kafka 命令行操作

![image-20260313175730302](./pictures/image-20260313175730302-3395851.png)

## 1、topic 主题命令

```bash
# 查看操作主题命令参数
bin/kafka-topics.sh
# 参数 描述
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

# 查看当前服务器中的所有 topic
bin/kafka-topics.sh --bootstrap-server hadoop000:9092 --list
# 创建 first topic
bin/kafka-topics.sh --bootstrap-server hadoop000:9092 --create
--partitions 1 --replication-factor 3 --topic first
# 查看 first 主题的详情
bin/kafka-topics.sh --bootstrap-server hadoop000:9092 --describe --topic first
# 修改分区数（注意：分区数只能增加，不能减少）
bin/kafka-topics.sh --bootstrap-server hadoop000:9092 --alter --topic first --partitions 3
# 创建 topic 时指定可靠性参数（生产常用）
bin/kafka-topics.sh --bootstrap-server hadoop000:9092 --create \
  --partitions 3 --replication-factor 3 --topic my-topic \
  --config min.insync.replicas=2 \
  --config retention.ms=604800000
# 已有 topic 动态修改参数
bin/kafka-configs.sh --bootstrap-server hadoop000:9092 --entity-type topics \
  --entity-name my-topic --alter --add-config min.insync.replicas=2
# 删除 topic
bin/kafka-topics.sh --bootstrap-server hadoop000:9092 --delete --topic first
```

## 2、producer 生产者命令

```bash
# 查看操作消费者命令参数
bin/kafka-console-producer.sh
# 参数 描述
--bootstrap-server <String: server toconnect to> 连接的 Kafka Broker 主机名称和端口号。
--topic <String: topic> 操作的 topic 名称。
# 发送消息
bin/kafka-console-producer.sh --bootstrap-server hadoop000:9092  --topic first
```

## 3、consumer 消费者命令

```bash
# 查看操作消费者命令参数
bin/kafka-console-consumer.sh
# 参数 描述
--bootstrap-server <String: server toconnect to> 连接的 Kafka Broker 主机名称和端口号。
--topic <String: topic> 操作的 topic 名称。
--from-beginning 从头开始消费。
--group <String: consumer group id> 指定消费者组名称。
# 消费 first 主题中的数据
bin/kafka-console-consumer.sh --bootstrap-server hadoop000:9092 --topic first
# 把主题中所有的数据都读取出来（包括历史数据）。
bin/kafka-console-consumer.sh --bootstrap-server hadoop000:9092 --from-beginning --topic first
```

