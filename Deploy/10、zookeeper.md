# 二进制包安装

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
mv /opt/zookeeper/conf/zoo_sample.cfg zoo.cfg

vi /opt/zookeeper/conf/zoo.cfg

# 集群数据存储目录
dataDir=/data/zookeeper/data

# 事务日志目录
dataLogDir=/data/zookeeper/logs

# 服务器监听端口
clientPort=2181

# 集群通信端口
peerPort=2888
electionPort=3888

# 集群成员配置（id与myid文件对应）
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
vi /etc/systemd/system/zookeeper.service

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
```

## 6、同步集群文件

```bash
xsync /opt/zookeeper
xsync /data/zookeeper
xsync /etc/profile.d/zookeeper.sh
xsync /etc/systemd/system/zookeeper.service

# 三台机器分别创建 myid 文件
echo "1" | sudo tee /data/zookeeper/data/myid
echo "2" | sudo tee /data/zookeeper/data/myid
echo "3" | sudo tee /data/zookeeper/data/myid

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
