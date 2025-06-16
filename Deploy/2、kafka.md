# 1、kafka 下载

[Kafka官网](https://kafka.apache.org/downloads) 中选择 Binary download 获取包。

3.0 版本以后建议使用 jdk11

2.8 版本以前使用jdk8

# 2、集群规划

规划为三台机器 hadoop000 hadoop001 hadoop002

```bash
cd /opt
# 解压安装包
tar -zxvf kafka_2.12-3.0.0.tgz
# 修改解压后的文件名称
mv kafka_2.12-3.0.0 kafka
```

# 3、配置文件修改

## 3.1、zookeeper 配置文件修改

```bash
# 创建目录文件夹
mkdir -p /data/zookeeper
# 修改 zookeeper.properties
vi /opt/kafka/config/zookeeper.properties
```

```bash
# 数据地址
dataDir=/data/zookeeper/data
# 日志文件路径
zookeeper.log.dir=/data/zookeeper/logs
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
```

```bash
# 在dataDir下创建myid文件 添加 1 2 3
# server.x 对应机器的myid
echo 1 > /data/zookeeper/myid
```

## 3.2、kafka 配置文件修改

```bash
# 创建目录文件夹
mkdir -p /data/kafka
# 修改 server.properties
vi /opt/kafka/config/server.properties
```

```bash
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

# 4、集群分发脚本

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
# 对脚本授权
chmod +755 xsync

# 进行集群分发
xsync module/kafka/
```

# 5、环境变量配置

```bash
# 创建 Kafka 环境变量配置文件
vi /etc/profile.d/my_env.sh
```

```bash
#KAFKA_HOME
export KAFKA_HOME=/opt/kafka
export PATH=$PATH:$KAFKA_HOME/bin
```

```bash
# 刷新环境变量
source /etc/profile

# 修改 kafka 内存占用
cd /opt/kafka/bin/
vi kafka-server-start.sh

# 找到此行并修改 32G 机器建议 8G
export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"

# 修改 zookeeper 内存占用
cd /opt/kafka/bin/
vi zookeeper-server-start.sh

# 找到此行并修改 32G 机器建议 2G
export KAFKA_HEAP_OPTS="-Xmx512M -Xms512M"
```

# 6、启动集群

## 6.1、启动 zookeeper

```bash
# 启动 zookeeper
/opt/kafka/bin/zookeeper-server-start.sh -daemon /opt/kafka/config/zookeeper.properties

# 关闭 zookeeper
/opt/kafka/bin/zookeeper-server-stop.sh

# 正常启动后 2181 3888 三台机器端口都会被占用，其中一台 2888 端口被占用，是 leader 节点。
```

## 6.2、启动 kafka

```bash
# 启动 kafka
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties

# 关闭 kafka
/opt/kafka/bin/kafka-server-stop.sh
```

## 6.3、注意事项

停止 kafka 集群时，一定要等 kafka 所有节点进程全部停止后再停止 zookeeper 集群。因为 zookeeper 集群当中记录着 kafka 集群相关信息，zookeeper 集群一旦先停止，kafka 集群就没有办法再获取停止进程的信息，只能手动杀死 kafka 进程了。

## 6.4、常见问题

```
//问题1
The Cluster ID VB7m6OM6SwS5oNJXn20XUA doesn't match stored clusterId
//解决1
删除 server.properties 中 log.dirs 的所有文件

//问题2
Cannot open channel to 2 at election address
//解决2
检查各节点的防火墙有没有关闭||检查各节点/etc/hosts内容是否一致
```

# 7、kafka 测试

## 7.1、topic

查看操作主题命令行参数

`kafka-topics.sh`

```
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

查看当前服务器中的所有 topic

`kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --list`

创建 first topic，1分区3副本

`kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --create --partitions 1 --replication-factor 3 --topic first`

查看 first 主题的详情

`kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --describe --topic first`

修改分区数（注意：分区数只能增加，不能减少）

`kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --alter --topic first --partitions 3`

删除 topic

`kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --delete --topic first`

## 7.2、producer

发送消息

`kafka-console-producer.sh --bootstrap-server 127.0.0.1:9092  --topic first`

## 7.3、consumer

查看操作消费者命令参数

`kafka-console-consumer.sh`

```
--bootstrap-server <String: server toconnect to> 连接的 Kafka Broker 主机名称和端口号。
--topic <String: topic> 操作的 topic 名称。
--from-beginning 从头开始消费。
--group <String: consumer group id> 指定消费者组名称。
```

消费 first 主题中的数据

`kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic first`

把主题中所有的数据都读取出来，包括历史数据

`kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --from-beginning --topic first`

## 7.4、consumer groups

查看消费者组命令命令

`kafka-consumer-groups.sh`

```
--bootstrap-server <String: server toconnect to> 连接的 Kafka Broker 主机名称和端口号。
--describe 查看主题详细描述。
--group <String: consumer group id> 指定消费者组名称。
```

查看所有的 group

`./kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --list`

查看 my-group 消费者组的偏移量

`./kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --group my-group --describe`

# 8、创建服务

User 和 Group 可以根据需要添加

zookeeper 服务

```bash
sudo tee /etc/systemd/system/zookeeper.service > /dev/null <<EOF
[Unit]
Description=Apache ZooKeeper Server
After=network.target

[Service]
Type=simple
User=zookeeper
Group=zookeeper
Environment=JAVA_HOME=/opt/jdk/jdk1.8.0_391
Environment=PATH=$JAVA_HOME/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-failure
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
EOF
```

kafka 服务

```bash
sudo tee  /etc/systemd/system/kafka.service > /dev/null <<EOF
[Unit]
Description=Apache Kafka Server
After=zookeeper.service

[Service]
Type=simple
User=kafka
Group=kafka
Environment=JAVA_HOME=/opt/jdk/jdk1.8.0_391
Environment=PATH=$JAVA_HOME/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
EOF
```

启动服务

```bash
# 重新加载 systemd
sudo systemctl daemon-reexec
sudo systemctl daemon-reload

# 启用 ZooKeeper 和 Kafka 服务（开机自动启动）
sudo systemctl enable zookeeper
sudo systemctl enable kafka

# 启动服务
sudo systemctl start zookeeper
sudo systemctl start kafka

# 查看状态
sudo systemctl status zookeeper
sudo systemctl status kafka
```

