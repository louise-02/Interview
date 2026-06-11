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

```sql
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

## 5、日志文件

| **表名**                           | **内容**                                           | **写入频率**           | **备注**                     | **可能占用情况**               |
| ---------------------------------- | -------------------------------------------------- | ---------------------- | ---------------------------- | ------------------------------ |
| **system.text_log**                | ClickHouse 内部运行日志（Debug/Trace/Information） | 高频（每个事件）       | Trace/Debug 不控制会快速膨胀 | ✅ 可能几十 GB~百 GB            |
| **system.query_log**               | SQL 查询执行信息                                   | 高频（每条查询）       | 保留天数越长越大             | ✅ 可能几十 GB                  |
| **system.part_log**                | MergeTree 各数据块/分区操作                        | 中等（每次写入/merge） | 对调试合并有用               | ✅ 可能数 GB~几十 GB            |
| **system.trace_log**               | 内部 Trace 事件                                    | 高频（低级事件）       | Trace 级别日志               | ✅ 高负载下可能超过 5 GB        |
| **system.metric_log**              | 系统指标（CPU/Memory/IO 等）                       | 定时                   | 按 interval 写入             | ✅ 高频指标+长时间保留可能数 GB |
| **system.asynchronous_metric_log** | 异步系统指标                                       | 定时                   | 类似 metric_log              | ❌ 一般 1~2 GB                  |
| **system.latency_log**             | 查询延迟分析                                       | 低频                   | 一般占用小                   | ❌ 一般 <1 GB                   |
| **system.processors_profile_log**  | 查询处理器级 profile                               | 高频（复杂查询）       | 大查询产生多行               | ✅ 高并发可能 >5 GB             |
| **system.query_metric_log**        | 查询指标日志                                       | 中低频                 | 一般占用小                   | ❌ <1 GB                        |
| **system.error_log**               | 错误日志                                           | 低频                   | 只记录错误                   | ❌ <1 GB                        |
| **system.os_logs**                 | 操作系统事件日志（可选）                           | 系统事件触发           | 一般占用小                   | ❌ <1 GB                        |

**日期文件级别修改**

```sql
vi /etc/clickhouse-server/config.xml

<text_log>
    <database>system</database>
    <table>text_log</table>
    <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    <max_size_rows>1048576</max_size_rows>
    <reserved_size_rows>8192</reserved_size_rows>
    <buffer_size_rows_flush_threshold>524288</buffer_size_rows_flush_threshold>
    <flush_on_crash>false</flush_on_crash>
    <level>information</level>
</text_log>
```

**日志文件清理**

```sql
-- 查看当前所有表占用情况
SELECT 
    database, 
    table, 
    formatReadableSize(sum(bytes_on_disk)) AS size,
    count() AS parts_count
FROM system.parts
WHERE active = 1
GROUP BY database, table
ORDER BY sum(bytes_on_disk) DESC;

-- 查看分区情况
SELECT
    database,
    table,
    partition,
    count() AS part_count,
    formatReadableSize(sum(bytes)) AS size,
    sum(rows) AS rows,
    min(min_time) AS start_time,
    max(max_time) AS end_time
FROM system.parts
WHERE 
table IN ('text_log', 'part_log')
and active
GROUP BY database, table, partition
ORDER BY sum(bytes) desc;

-- 删除历史分区（例如删除 2025-07 月份的日志）
ALTER TABLE system.text_log DROP PARTITION '202507';
ALTER TABLE system.part_log DROP PARTITION '202508';
-- 如果文件过大 需要跳过校验
ALTER TABLE system.text_log DROP PARTITION '202508' SETTINGS max_partition_size_to_drop = 0;
ALTER TABLE system.part_log DROP PARTITION '202508' SETTINGS max_partition_size_to_drop = 0;

-- 设置日志表只保存15天
ALTER TABLE system.text_log
MODIFY TTL event_time + INTERVAL 15 DAY;

-- 验证是否修改成功
SHOW CREATE TABLE system.text_log;
```

## 6、数据备份和恢复

两台服务器上都安装 clickhouse-backup

```bash
# 在两台服务器上执行
wget https://github.com/AlexAkulov/clickhouse-backup/releases/download/v2.4.2/clickhouse-backup-linux-amd64.tar.gz

tar -xzvf clickhouse-backup-linux-amd64.tar.gz

cd ./build/linux/amd64

sudo mv clickhouse-backup /usr/local/bin/

sudo chmod +x /usr/local/bin/clickhouse-backup

# 数据会备份在 data_path/backup 下
sudo mkdir -p /data/clickhouse/data/backup
sudo chown -R clickhouse:clickhouse /data/clickhouse/data/backup
sudo chmod 755 /data/clickhouse/data/backup
```

备份服务器配置文件

```bash
# vi /etc/clickhouse-backup/config.yml - 备份服务器专用配置
# 适用于：zd_lbs(32GB) + zd_cermp(8GB) 总共约40GB数据

general:
  # 存储类型：不使用远程存储，通过文件复制方式传输
  remote_storage: none
  # 本地保留备份数量：保留3个备份方便轮转检查
  backups_to_keep_local: 3
  # 分片备份：启用后大表会分成多个文件，避免内存溢出
  backup_sharding: true
  # 单个分片文件最大大小：1GB一个文件，方便传输和管理
  max_file_size: 1073741824
  # 禁用进度条：减少日志输出，提高备份效率
  disable_progress_bar: true
  # 集群恢复设置：单机环境保持为空
  restore_schema_on_cluster: ""

clickhouse:
  # 连接用户名：建议使用专用备份用户
  username: "backup_user"
  # 连接密码：备份用户的密码
  password: "secure_password"
  # 服务器地址：本地localhost
  host: "localhost"
  # 服务端口：默认ClickHouse端口
  port: 9000
  # SSL安全连接：生产环境建议开启，此处为兼容性设为false
  secure: false
  # 跳过证书验证：开发环境可使用，生产环境建议关闭
  skip_verify: false
  # ClickHouse数据目录路径：默认路径，通常不需要修改 默认是/var/lib/clickhouse/
  data_path: "/data/clickhouse/data/"
  # 跳过备份的表：系统表和临时表不需要备份
  skip_tables:
    - "system.*"                    # 跳过所有系统表
    - "INFORMATION_SCHEMA.*"        # 跳过信息模式表
    - "information_schema.*"        # 跳过信息模式表（兼容不同版本）
    - "*.tmp_*"                     # 跳过临时表
    - "*.backup_*"                  # 跳过备份过程中产生的临时表
  # 备份超时时间：2小时，适应您的40GB数据量
  timeout: 2h
  # 按分区冻结：启用后按分区逐个冻结，减少锁表时间
  freeze_by_part: true
  # 复制表同步：设为false避免等待其他副本，加快备份速度
  sync_replicated_tables: false
  # 日志级别：info级别提供必要的操作信息
  log_level: "info"
  # 分布式DDL超时：2分钟，单机环境影响不大
  distributed_ddl_task_timeout: 120

# 压缩配置：平衡压缩比和CPU消耗
compression:
  # 压缩格式：tar格式兼容性好，压缩效率适中
  format: tar
  # 压缩级别：2级压缩，在速度和压缩比间取得平衡
  level: 2
  # 最小压缩文件大小：10MB以下文件不压缩，避免小文件压缩开销
  min_bytes: 10485760
```

```bash
general:
  remote_storage: none
  backups_to_keep_local: 3
  backup_sharding: true
  max_file_size: 1073741824
  disable_progress_bar: true
  restore_schema_on_cluster: ""

clickhouse:
  username: "backup_user"
  password: "secure_password"
  host: "localhost"
  port: 9000
  secure: false
  skip_verify: false
  data_path: "/data/clickhouse/data/"
  skip_tables:
    - "system.*"
    - "INFORMATION_SCHEMA.*"
    - "information_schema.*"
    - "*.tmp_*"
    - "*.backup_*"
  timeout: 2h
  freeze_by_part: true
  sync_replicated_tables: false
  log_level: "info"
  distributed_ddl_task_timeout: 120

compression:
  format: tar
  level: 2
  min_bytes: 10485760
```

恢复服务器配置文件

```bash
# vi /etc/clickhouse-backup/config.yml - 恢复服务器专用配置
# 专门用于从备份文件恢复数据

general:
  # 存储类型：不使用远程存储，从本地备份文件恢复
  remote_storage: none
  # 本地保留备份数量：恢复服务器只需保留当前使用的备份
  backups_to_keep_local: 1
  # 分片备份：恢复时不需要分片处理
  backup_sharding: false
  # 禁用进度条：减少日志输出，保持界面简洁
  disable_progress_bar: true
  # 集群恢复设置：单机恢复保持为空
  restore_schema_on_cluster: ""

clickhouse:
  # 连接用户名：恢复时可以使用default用户或专用用户
  username: "default"
  # 连接密码：根据实际配置填写
  password: ""
  # 服务器地址：本地localhost
  host: "localhost"
  # 服务端口：默认ClickHouse端口
  port: 9000
  # SSL安全连接：根据恢复服务器配置决定
  secure: false
  # 跳过证书验证：恢复服务器通常为可信环境
  skip_verify: false
  # ClickHouse数据目录路径：保持与备份服务器一致 默认是/var/lib/clickhouse/
  data_path: "/data/clickhouse/data/"

  # 跳过恢复的表：恢复时通常不需要跳过任何表
  skip_tables: []
  # 恢复时保留空数组，恢复所有传输过来的表
       
  # 恢复超时时间：3小时，恢复可能比备份耗时更长
  timeout: 3h
  # 按分区冻结：恢复时不需要冻结操作
  freeze_by_part: false
  # 复制表同步：恢复时设为false避免依赖其他副本
  sync_replicated_tables: false
  # 日志级别：info级别便于监控恢复进度
  log_level: "info"
  # 分布式DDL超时：恢复时适当延长超时时间
  distributed_ddl_task_timeout: 180

# 压缩配置：恢复时自动识别备份文件的压缩格式
compression:
  # 压缩格式：与备份配置保持一致
  format: tar
  # 压缩级别：恢复时压缩级别不影响操作
  level: 1
  # 最小压缩文件大小：恢复时此参数不生效
  min_bytes: 0
```

```bash
general:
  remote_storage: none
  backups_to_keep_local: 1
  backup_sharding: false
  disable_progress_bar: true
  restore_schema_on_cluster: ""

clickhouse:
  username: "default"
  password: ""
  host: "localhost"
  port: 9000
  secure: false
  skip_verify: false
  data_path: "/data/clickhouse/data/"
  skip_tables: []
  timeout: 3h
  freeze_by_part: false
  sync_replicated_tables: false
  log_level: "info"
  distributed_ddl_task_timeout: 180

compression:
  format: tar
  level: 1
  min_bytes: 0
```

验证配置文件

```bash
# 查看当前有效配置
clickhouse-backup print-config

# 测试配置连接
clickhouse-backup test-config
```

创建全量备份

```bash
# 在源服务器创建备份
clickhouse-backup create full_backup_$(date +%Y%m%d)

# 查看备份文件位置
clickhouse-backup list
```

全量备份高级选项

```bash
# 1. 只备份表结构，不备份数据
clickhouse-backup create --schema-only backup_name

# 2. 备份时压缩数据
clickhouse-backup create --compress backup_name

# 3. 指定压缩级别（1-9，默认1）
clickhouse-backup create --compress-level=6 backup_name

# 4. 并行备份（提高速度）
clickhouse-backup create --parallel=4 backup_name

# 5. 备份特定数据库
clickhouse-backup create --tables=zd_cermp.* backup_name

# 6. 排除特定数据库
clickhouse-backup create --exclude-databases=system backup_name

# 7. 备份特定表
clickhouse-backup create --tables=zd_lbs.table1,zd_cermp.table2 backup_name

# 8. 跳过表数据，只备份结构
clickhouse-backup create --skip-table-data='*.log_*,temp_*' backup_name

# 9. 设置备份描述
clickhouse-backup create --description="生产环境全量备份" backup_name
```

恢复全量备份

```bash
# 查看可用的备份
clickhouse-backup list

# 执行恢复（这会覆盖现有数据）
clickhouse-backup restore full_backup_20241201

# 查看恢复进度（通过ClickHouse日志）
sudo tail -f /var/log/clickhouse-server/clickhouse-server.log
```

高级恢复选项

```bash
# 1. 只恢复表结构（不恢复数据）
clickhouse-backup restore --schema only full_backup_20241201

# 2. 恢复特定数据库
clickhouse-backup restore --tables default.* full_backup_20241201

# 3. 排除特定表
clickhouse-backup restore --exclude-databases system full_backup_20241201

# 4. 干运行（预览恢复操作）
clickhouse-backup restore --dry-run full_backup_20241201

# 跳过复制表
--skip-replicated-tables
# 数据库映射
--restore-database-mapping=zd_cermp:new_zd_cermp
```

数据一致性校验

```sql
-- 在目标服务器执行，与源服务器对比
-- 1. 检查数据库和表
SHOW DATABASES;
SHOW TABLES FROM zd_lbs;
SHOW TABLES FROM zd_cermp;

-- 2. 检查表行数
SELECT 
    database,
    name,
    total_rows,
    formatReadableSize(total_bytes) as size
FROM system.tables 
WHERE database IN ('zd_lbs', 'zd_cermp', 'default')
ORDER BY total_rows DESC;

-- 3. 抽样检查数据
SELECT count(*) FROM zd_lbs.你的大表名 LIMIT 1;
SELECT count(*) FROM zd_cermp.你的大表名 LIMIT 1;
```

