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
# 下载 rsync
yum install -y rsync
# 对脚本授权
chmod +755 /usr/bin/xsync

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





