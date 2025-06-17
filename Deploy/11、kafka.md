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

## 5、创建 systemd 服务

```bash
vi /etc/systemd/system/kafka.service

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



# Kafka

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

