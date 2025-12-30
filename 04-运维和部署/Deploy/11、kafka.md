# Kafka 简介

[Kafka官网](https://kafka.apache.org/downloads) 中选择 Binary download 获取包。

3.0 版本以后建议使用 jdk11

2.8 版本以前使用jdk8

# 二进制包安装

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

num.partitions=3
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
log.cleaner.enable=true

zookeeper.connect=192.168.1.101:2181,192.168.1.102:2181,192.168.1.103:2181/kafka
zookeeper.connection.timeout.ms=18000

num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

group.initial.rebalance.delay.ms=0
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

# Kafka 配置

## 1、日志地址修改

```bash
vi /opt/kafka/bin/kafka-run-class.sh

# 修改此处
if [ "x$LOG_DIR" = "x" ]; then
  LOG_DIR="$base_dir/logs"
fi
# 修改为以下内容
LOG_DIR="/data/kafka/logs"
```

## 2、堆内存修改

```bash
vi /opt/kafka/bin/kafka-server-start.sh

if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
fi
```

## 3、配置文件

```bash
vi /opt/kafka/config/server.properties

#broker 的全局唯一编号，不能重复，只能是数字。每台机器不一样 三台机器可以设置为 1 2 3
broker.id=1
#处理网络请求的线程数量
num.network.threads=3
#用来处理磁盘 IO 的线程数量
num.io.threads=8
#发送套接字的缓冲区大小
socket.send.buffer.bytes=102400
#接收套接字的缓冲区大小
socket.receive.buffer.bytes=102400
#请求套接字的缓冲区大小
socket.request.max.bytes=104857600
#kafka 运行日志(数据)存放的路径，路径不需要提前创建，kafka 自动帮你创建，可以配置多个磁盘路径，路径与路径之间可以用"，"分隔
log.dirs=/data/kafka

#topic 在当前 broker 上的分区个数
num.partitions=1
#用来恢复和清理 data 下数据的线程数量
num.recovery.threads.per.data.dir=1
# 每个 topic 创建时的副本数，默认时 1 个副本
offsets.topic.replication.factor=1

#segment 文件保留的最长时间，超时将被删除
log.retention.hours=168
#每个 segment 文件的大小，默认最大 1G
log.segment.bytes=1073741824
# 检查过期数据的时间，默认 5 分钟检查一次是否数据过期
log.retention.check.interval.ms=300000
# 监听地址
listeners=PLAINTEXT://0.0.0.0:9092
# 对外公布的地址 ip为本机真实ip
advertised.listeners=PLAINTEXT://ip:9092
#配置连接 Zookeeper 集群地址（在 zk 根目录下创建/kafka，方便管理）
zookeeper.connect=hadoop102:2181,hadoop103:2181,hadoop104:2181/kafka
```

对于分区的配置

```bash
# topic 在当前 broker 上的分区个数
num.partitions=9

# 内部消费者偏移主题副本数
offsets.topic.replication.factor=3
# 事务状态日志主题副本数
transaction.state.log.replication.factor=3
# 事务状态日志主题最小同步副本确认数
transaction.state.log.min.isr=2
```



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

