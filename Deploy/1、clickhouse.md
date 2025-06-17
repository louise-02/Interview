# yum 安装

## 1、安装

添加官方仓库

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
```

查看可用版本

```bash
yum --showduplicates list clickhouse-server
# 此处通过 github 获得25.3.3.42是最新的LTS版本版本
yum --showduplicates list clickhouse-server | grep 25.3.3.42
```

安装 ClickHouse 服务器和客户端

```bash
sudo yum install -y clickhouse-server clickhouse-client
# 指定版本号
sudo yum install -y clickhouse-server-25.3.3.42-1 clickhouse-client-25.3.3.42-1
```

3、启动 ClickHouse 服务器

```bash
sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
sudo systemctl status clickhouse-server
clickhouse-client # 或 "clickhouse-client --password" 如果您设置了密码。
```

4、配置远程访问

默认仅允许本地连接。若需远程访问，修改配置文件：

```bash
sudo vim /etc/clickhouse-server/config.xml
```

找到 `<listen_host>` 配置项，取消注释并修改为：

```xml
<listen_host>0.0.0.0</listen_host>
<interserver_http_host>节点真实IP</interserver_http_host>
```

重启服务生效：

```bash
sudo systemctl restart clickhouse-server
```

开放防火墙端口

```bash
sudo firewall-cmd --permanent --add-port=8123/tcp  # HTTP API 端口
sudo firewall-cmd --permanent --add-port=9000/tcp  # 客户端TCP端口
sudo firewall-cmd --permanent --add-port=9009/tcp  # 跨服务器复制端口
sudo firewall-cmd --reload
```

5、测试连接

```bash
clickhouse-client -h 127.0.0.1 --port 9000 -u default --password

SELECT 1;
```

## 2、卸载

关闭服务

```bash
sudo systemctl stop clickhouse-server
sudo systemctl disable clickhouse-server
```

卸载

```bash
sudo yum remove clickhouse-server clickhouse-client
```

删除文件

```bash
sudo rm -rf /var/lib/clickhouse /var/log/clickhouse-server /etc/clickhouse-server /etc/clickhouse-client
```

# clickhouse

## 1、yum 安装目录

| 类型               | 路径                                               | 说明                                                         |
| ------------------ | -------------------------------------------------- | ------------------------------------------------------------ |
| 📄 主程序           | `/usr/bin/clickhouse-server`                       | ClickHouse 服务器主程序执行文件                              |
| 📄 客户端程序       | `/usr/bin/clickhouse-client`                       | ClickHouse 客户端命令行工具                                  |
| 📁 配置文件         | `/etc/clickhouse-server/`                          | 主配置目录，包含 `config.xml`、`users.xml` 等配置            |
| 📄 **主配置文件**   | `/etc/clickhouse-server/config.xml`                | ClickHouse 主配置文件，包含网络、端口、路径等设置            |
| 📄 **用户配置文件** | `/etc/clickhouse-server/users.xml`                 | 管理用户权限、配额等                                         |
| 📁 **数据目录**     | `/var/lib/clickhouse/`                             | 默认的数据存储位置，包括表数据、元数据等                     |
| 📁 **日志目录**     | `/var/log/clickhouse-server/`                      | 存放 ClickHouse 的运行日志、错误日志等                       |
| 📁 服务控制文件     | `/etc/systemd/system/clickhouse-server.service`    | ClickHouse 的 systemd 服务文件（有时在 `/usr/lib/systemd/...`） |
| 📁 临时目录         | `/var/lib/clickhouse/tmp/`                         | ClickHouse 执行中用到的临时目录                              |
| 📁 **表结构元数据** | `/var/lib/clickhouse/metadata/`                    | 存放数据库和表的结构定义                                     |
| 📁 动态库目录       | `/usr/lib/clickhouse/` 或 `/usr/lib64/clickhouse/` | ClickHouse 依赖的共享库（按系统架构不同路径可能略有差异）    |

## 2、配置文件

用户名和密码设置

```bash
vi /etc/clickhouse-server/users.xml

# 设置默认用户名的密码
<users>
    <default>
        <password>yourpassword</password>
        ...
    </default>
</users>

# 例如设置为 Qx93@dV!zLp6#nRg
echo -n 'Qx93@dV!zLp6#nRg' | sha256sum
# 生成 983ade067c8a796f4d1f12e9d2a19a45a7acd35654afffe52f953101c1355ec0  -
<users>
    <default>
        <password_sha256_hex>983ade067c8a796f4d1f12e9d2a19a45a7acd35654afffe52f953101c1355ec0</password_sha256_hex>
        <networks>
            <ip>::/0</ip>  <!-- 根据需要限制为某些网段 -->
        </networks>
        <profile>default</profile>
        <quota>default</quota>
    </default>
</users>
```

数据文件存储路径设置

```bash
vi /etc/clickhouse-server/config.xml

# 主数据存储路径
<path>/var/lib/clickhouse/</path>
# 临时文件路径
<tmp_path>/var/lib/clickhouse/tmp/</tmp_path>
# 用户文件目录
<user_files_path>/var/lib/clickhouse/user_files/</user_files_path>
# 格式定义目录
<format_schema_path>/var/lib/clickhouse/format_schemas/</format_schema_path>

# 可更改为以下目录
<path>/data/clickhouse/data/</path>
<tmp_path>/data/clickhouse/tmp/</tmp_path>

# 授权
sudo mkdir -p /data/clickhouse/data
sudo mkdir -p /data/clickhouse/tmp
sudo chown -R clickhouse:clickhouse /data/clickhouse
sudo chmod -R 755 /data/clickhouse
```

## 3、新建用户授权

```sql
# 创建数据库
CREATE DATABASE IF NOT EXISTS my_database;

# 创建用户
CREATE USER IF NOT EXISTS my_user
IDENTIFIED WITH plaintext_password BY 'my_password';

# 授权
GRANT ALL ON my_database.* TO my_user;
```

## 4、复制表配置

zookeeper 配置

```
vim /etc/clickhouse-server/config.xml
 
   <zookeeper>
        <node>
            <host>10.168.106.107</host>
            <port>2181</port>
        </node>
        <node>
            <host>10.168.106.108</host>
            <port>2181</port>
        </node>
        <node>
            <host>10.168.106.109</host>
            <port>2181</port>
        </node>
    </zookeeper>
```

所有节点创建集群配置文件

```xml
vi /etc/clickhouse-server/config.d/clusters.xml

<yandex>
    <remote_servers>
        <!-- 定义集群名称（如 full_replica_cluster） -->
        <full_replica_cluster>
            <shard>  <!-- 只有一个分片 -->
                <internal_replication>true</internal_replication>
                <replica>
                    <host>节点1的IP</host>  <!-- 替换为实际IP或主机名 -->
                    <port>9000</port>
                </replica>
                <replica>
                    <host>节点2的IP</host>  <!-- 替换为实际IP或主机名 -->
                    <port>9000</port>
                </replica>
            </shard>
        </full_replica_cluster>
    </remote_servers>
</yandex>
```

两个节点分别创建宏配置文件

```
vi /etc/clickhouse-server/config.d/macros.xml

<yandex>
    <macros>
        <shard>01</shard>          <!-- 所有节点相同分片ID -->
        <replica>node1</replica>   <!-- 节点唯一标识 -->
    </macros>
</yandex>

<yandex>
    <macros>
        <shard>01</shard>          <!-- 分片ID与节点1相同 -->
        <replica>node2</replica>   <!-- 不同副本标识 -->
    </macros>
</yandex>
```

设置文件权限

```
sudo chown clickhouse:clickhouse /etc/clickhouse-server/config.d/*.xml
sudo chmod 644 /etc/clickhouse-server/config.d/*.xml
```

重启 clickhouse

```
sudo systemctl restart clickhouse-server
# 检查状态
sudo systemctl status clickhouse-server
```

查看是否生效

```
-- 检查集群配置
SELECT cluster, shard_num, host_name, replica_num FROM system.clusters;

-- 检查宏变量
SELECT * FROM system.macros;

-- 测试创建复制表（在任一节点执行）
CREATE TABLE default.replica_test ON CLUSTER full_replica_cluster (id Int32)
ENGINE = ReplicatedMergeTree ORDER BY id;

-- 检查复制队列状态
SELECT 
    type,
    create_time,
    is_blocked,
    error
FROM system.replication_queue
WHERE table = 'replica_test'
ORDER BY create_time DESC
LIMIT 5;
```

