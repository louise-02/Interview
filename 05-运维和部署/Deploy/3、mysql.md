# MySQL 简介

[MySQL 官网](https://www.mysql.com) | [文档](https://dev.mysql.com/doc/)

MySQL 是最常用的**开源关系型数据库**，适用于业务库、主从复制、读写分离等场景。

| 功能 | 说明 |
| ---- | ---- |
| InnoDB | 默认存储引擎，支持事务 |
| 主从复制 | 读写分离、高可用基础 |
| 二进制日志 | 备份、增量同步、审计 |

常用端口：

| 端口 | 说明 |
| ---- | ---- |
| 3306 | 客户端连接 |

## 部署方式推荐

| 环境 | 推荐方式 | 说明 |
| ---- | -------- | ---- |
| **生产** | **yum** | 核心库建议原生安装，便于备份、主从与性能调优 |
| 开发 / 测试 | Docker | 快速搭建，非核心业务库可用 |
| 离线环境 | yum 离线 rpm 包 | 见下文离线安装章节 |

---

# docker 安装（MySQL 8.0）

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)  
> 偏生产检查清单见 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md)

## 1、创建目录

```bash
mkdir -p /data/docker/mysql/{conf,data,logs}
```

## 2、偏生产配置（挂载）

开发测试可跳过，直接启动；偏生产建议挂载 `my.cnf`（**首次 `up` 前**复制并改内存参数）。

```bash
cd /path/to/Deploy/docker
cp conf/mysql/my.cnf /data/docker/mysql/conf/
vi /data/docker/mysql/conf/my.cnf   # 按宿主机内存调整 innodb_buffer_pool_size
```

`innodb_buffer_pool_size`、慢日志、binlog 等说明见下文 [MySQL 配置与运维](#mysql-配置与运维)。容器内路径已在示例中写好（`log-error`、`slow_query_log_file` 指向 `/var/log/mysql/`）。

同时修改 [docker/mysql.yml](./docker/mysql.yml) 中 `MYSQL_ROOT_PASSWORD`，勿用默认弱密码。

## 3、启动

```bash
docker compose -f mysql.yml up -d
```

## 4、初始化

```bash
docker exec -it mysql mysql -uroot -p
# 创建业务库独立账号，勿长期用 root；见配置与运维「初始化与账号」
```

## 5、常用操作

```bash
docker compose -f mysql.yml ps
docker compose -f mysql.yml logs -f
docker compose -f mysql.yml restart
docker compose -f mysql.yml down
```

# yum 安装（MySQL 8.0.37 / 离线 8.0.45）

## 1、安装

```bash
# 卸载系统自带的 MariaDB（如果有）
sudo yum remove mariadb* -y

# 下载并添加 MySQL 官方 YUM 仓库
wget https://dev.mysql.com/get/mysql80-community-release-el7-5.noarch.rpm
sudo rpm -ivh mysql80-community-release-el7-5.noarch.rpm
sudo yum makecache

# 查看可用版本
yum --showduplicates list mysql-community-server | grep 8.0
# 可以看到输出
mysql-community-server.x86_64 8.0.37-1.el7  mysql80-community

# 安装指定版本 如8.0.37
sudo yum install mysql-community-server-8.0.37-1.el7 -y
# 也可以指定所有组件版本 不指定其他会用其他版本
sudo yum install mysql-community-client-8.0.37-1.el7 \
                 mysql-community-common-8.0.37-1.el7 \
                 mysql-community-libs-8.0.37-1.el7 \
                 mysql-community-server-8.0.37-1.el7 -y

# 启动数据库
sudo systemctl start mysqld
sudo systemctl enable mysqld
```

其他事项

```bash
# 如遇到 GPG 密钥配置报错
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql


# 防止 YUM 升级覆盖指定版本
vi /etc/yum.conf
exclude=mysql*
```

## 2、离线安装

```bash
# 官方 MySQL RPM Bundle 下载页面
https://dev.mysql.com/downloads/mysql/

# Red Hat Enterprise Linux 7 / Oracle Linux 7 (x86, 64-bit)

# 解压 RPM Bundle
tar -xvf mysql-8.0.45-1.el7.x86_64.rpm-bundle.tar

# 安装顺序
rpm -ivh \
mysql-community-common-8.0.45-1.el7.x86_64.rpm \
mysql-community-icu-data-files-8.0.45-1.el7.x86_64.rpm \
mysql-community-libs-8.0.45-1.el7.x86_64.rpm \
mysql-community-libs-compat-8.0.45-1.el7.x86_64.rpm \
mysql-community-client-plugins-8.0.45-1.el7.x86_64.rpm \
mysql-community-client-8.0.45-1.el7.x86_64.rpm \
mysql-community-server-8.0.45-1.el7.x86_64.rpm


yum localinstall -y *.rpm

# 启动 MySQL
systemctl enable mysqld
systemctl start mysqld
```

## 3、卸载

```bash
# 停止服务
sudo systemctl stop mysqld

# 卸载所有 MySQL 相关的 RPM 包
sudo yum remove -y mysql mysql-server mysql-libs mysql-common mysql-community-*

# 删除残留配置和数据文件
sudo rm -rf /var/lib/mysql
sudo rm -rf /etc/my.cnf
sudo rm -rf /etc/my.cnf.d
sudo rm -rf /var/log/mysqld.log
sudo rm -rf /usr/lib64/mysql
sudo rm -rf /usr/share/mysql

# 清理用户和组
sudo userdel -r mysql
sudo groupdel mysql
```

# MySQL 配置与运维

安装完成后：查目录 → 改生产配置 → 初始化账号 → 备份巡检。Docker 安装的 `my.cnf` 放 `/data/docker/mysql/conf/`，可参考下文模板。

## 1、安装目录

| 类型                         | 路径                                       | 说明                                       |
| ---------------------------- | ------------------------------------------ | ------------------------------------------ |
| 📄 主程序                     | `/usr/sbin/mysqld`                         | MySQL Server 可执行文件                    |
| 📁 配置文件                   | `/etc/my.cnf`                              | MySQL 的主配置文件                         |
| 📁 数据目录（首次启动后创建） | `/var/lib/mysql/`                          | 存放数据库数据、表、用户信息等             |
| 📁 日志目录                   | `/var/log/mysqld.log`                      | MySQL 启动及运行日志（包括初始 root 密码） |
| 📁 慢查询日志                 | `/var/log/mysql-slow.log`                  | 需配置 `slow_query_log=1`                  |
| 📁 系统服务文件               | `/usr/lib/systemd/system/mysqld.service`   | 用于 systemd 启动 MySQL                    |
| 📁 客户端工具                 | `/usr/bin/mysql`、`/usr/bin/mysqladmin` 等 | MySQL 客户端及管理工具                     |
| 📁 库文件                     | `/usr/lib64/mysql/`                        | MySQL 相关动态库                           |
| 📁 通用文件                   | `/usr/share/mysql/`                        | 包括错误信息、字符集、SQL 脚本等           |
| 📁 PID文件                    | `/var/run/mysqld/mysqld.pid`               | 进程ID文件                                 |

## 2、生产配置

以下示例按 **16GB 物理内存、专用 MySQL 节点** 编写。各内存项占物理内存的**参考比例**：

| 参数 | 建议占物理内存 | 16GB 示例 | 说明 |
| ---- | -------------- | --------- | ---- |
| `innodb_buffer_pool_size` | **60%～70%**（专用）；混部 **40%～50%** | `10G`（约 62%） | 最重要，缓存数据页与索引 |
| `innodb_log_buffer_size` | **0.5%～1%** | `64M` | 写 redo 的内存缓冲 |
| `tmp_table_size` + `max_heap_table_size` | 合计约 **2%** | 各 `128M` | 内存临时表上限（按会话） |
| `sort_buffer_size` / `join_buffer_size` | 单连接固定值 | 各 `8M` | **注意**：理论峰值 ≈ `max_connections × (sort + join)`，500 连接约 8GB，勿盲目调大 |

`innodb_log_file_size` 是**磁盘上** redo 文件大小（非 Buffer Pool），单文件常见 **512MB～1GB**，两组共约 1～2GB 磁盘即可。

```bash
chown -R mysql:mysql /data/mysql
chmod 750 /data/mysql
sudo mkdir -p /var/log/mysql
sudo chown -R mysql:mysql /var/log/mysql
vi /etc/my.cnf
```

```bash
[mysqld]
# ======================================
# 基础配置
# ======================================
user = mysql
port = 3306
basedir = /usr
datadir = /data/mysql
socket = /data/mysql/mysql.sock
pid-file = /var/run/mysqld/mysqld.pid
default-storage-engine = InnoDB

# ======================================
# 编码与时区
# ======================================
character-set-server = utf8mb4
collation-server = utf8mb4_general_ci
init_connect='SET NAMES utf8mb4'
default-time-zone = '+08:00'

# ======================================
# 连接相关
# ======================================
# 最大连接数
max_connections = 500
# 出错后允许最大重试连接数
max_connect_errors = 10000
# 非交互式连接空闲超时 应用程序连接
wait_timeout = 1800
# 交互式连接空闲超时 mysql命令行
interactive_timeout = 1800

# ======================================
# SQL 模式控制
# ======================================
# STRICT_TRANS_TABLES 开启严格数据校验，拒绝非法数据插入，防止插入超长字符串、非法日期等
# NO_ENGINE_SUBSTITUTION 禁止存储引擎替换，确保存储引擎按预期使用，避免自动降级
sql_mode=STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION

# ======================================
# 慢查询日志（用于定位性能瓶颈）
# ======================================
# 错误日志
log-error = /var/log/mysql/error.log
# 是否开启慢查询日志 1 表示开启，0 表示关闭
slow_query_log = 1
# 慢查询日志
slow_query_log_file = /var/log/mysql/slow.log
# 超过2秒即为慢查询
long_query_time = 2
# 没用索引的SQL也记录
log_queries_not_using_indexes = 1

# ======================================
# 二进制日志（备份、主从、审计）
# ======================================
# 主从唯一标识（必须配置）
server-id = 1
# bin log日志
log-bin = /var/log/mysql/mysql-bin
# 行格式（推荐）
binlog_format = row
# 日志保留 7 天（MySQL 8.0 用 binlog_expire_logs_seconds，expire_logs_days 已废弃）
binlog_expire_logs_seconds = 604800
# 每次事务同步（主从一致性高）
sync-binlog = 1

# ======================================
# 表结构
# ======================================
# 0 大小写敏感 Linux/Unix 默认
# 1 不区分大小写（存储小写） Windows/macOS 常用
# 2 不区分大小写（保留原样） macOS（老版本）
lower_case_table_names = 1

# ======================================
# 内存相关配置（重点优化）
# ======================================
# InnoDB Buffer Pool：用于缓存表数据和索引（最重要参数）
# 建议为系统内存的 60~70%（16GB × 65% ≈ 10GB）
innodb_buffer_pool_size = 10G
# InnoDB Log：磁盘上的 redo 文件大小（单文件），非 Buffer Pool 内存
# 写入量大时单文件 512MB～1GB，两组 innodb_log_files_in_group=2 共约 1～2GB 磁盘
innodb_log_file_size = 1G
# 日志文件数量（共用2GB日志空间）
innodb_log_files_in_group = 2
# Log Buffer：事务日志写入缓冲区
# 建议占用内存 0.5% 左右
innodb_log_buffer_size = 64M
# 控制写入性能与可靠性
# 每次提交刷盘（事务安全）建议保留 1
innodb_flush_log_at_trx_commit = 1
# 减少 double buffering 提升性能
innodb_flush_method = O_DIRECT
# 每张表独立存储（推荐）
innodb_file_per_table = 1


# ======================================
# 临时表/排序/连接缓存
# ======================================
# 内存临时表最大值（提高避免磁盘临时表） tmp_table_size + max_heap_table_size 建议占用内存 2% 左右
tmp_table_size = 128M
# 内存表最大值（与 tmp_table_size 保持一致）
max_heap_table_size = 128M
# 每连接排序缓冲区（占用内存 = sort_buffer × 并发）
sort_buffer_size = 8M
# 每连接 join 缓冲区（同上）
join_buffer_size = 8M

# ======================================
# 性能统计（建议开启）
# ======================================
# 性能监控支持
performance_schema = ON

[client]
socket=/data/mysql/mysql.sock
```

## 3、系统参数

MySQL 连接数较高时需调大文件描述符，通用模板见 [0、readme.md](./0、readme.md#系统参数生产通用)。

```bash
# /etc/security/limits.conf
mysql soft nofile 65536
mysql hard nofile 65536

# /usr/lib/systemd/system/mysqld.service 的 [Service] 段
LimitNOFILE=65536
```

## 4、初始化与账号

修改密码，配置权限：

```sql
# 获取默认 root 密码
sudo grep 'temporary password' /var/log/mysqld.log
# 登录
mysql -uroot -p

ALTER USER 'root'@'localhost' IDENTIFIED BY 'Aa123456..!';

use mysql;

update user set host='%' where user='root';

# 授权 root 拥有所有数据库的所有权限
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;

GRANT SYSTEM_USER ON *.* TO 'root'@'%';

FLUSH PRIVILEGES;

# 创建普通用户
CREATE USER 'cermp'@'%' IDENTIFIED BY 'P5x@jN2qZ4wS6';
GRANT ALL PRIVILEGES ON *.* TO 'cermp'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

生产环境建议业务库使用**独立账号 + 最小权限**，避免应用直连 root：

```sql
CREATE USER 'app_user'@'%' IDENTIFIED BY '强密码';
GRANT SELECT, INSERT, UPDATE, DELETE ON your_db.* TO 'app_user'@'%';
FLUSH PRIVILEGES;
DELETE FROM mysql.user WHERE User='';
DROP DATABASE IF EXISTS test;
```

## 5、备份与巡检

```bash
mysqldump -u root -p --single-transaction --routines --triggers your_db > /backup/your_db_$(date +%F).sql

mysql -e "SHOW STATUS LIKE 'Threads_connected';"
mysql -e "SHOW VARIABLES LIKE 'slow_query_log%';"
mysql -e "SHOW REPLICA STATUS\G"   # 主从环境
```

## 6、防火墙

```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

# 主从复制

## 主服务器

1、主服务器修改配置

```bash
vi /etc/my.cnf
```

修改以下内容

```bash
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-do-db = your_database
```

`server-id` 是一个唯一的 ID，用于区分不同的 MySQL 实例。主服务器通常设置为 1。

`log-bin` 启用二进制日志，这是主从复制的核心。

`binlog-do-db` 可选，如果你只想复制特定数据库的内容，可以指定，不知道默认复制全库。

2、重启 MySQL 服务

```bash
sudo systemctl restart mysqld
```

3、创建复制用户

```sql
CREATE USER 'replica_user'@'%' IDENTIFIED WITH mysql_native_password BY 'Aa123456..!';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';
FLUSH PRIVILEGES;
```

4、记录主服务器的二进制日志位置

```sql
SHOW MASTER STATUS;
```

`File` 对应从库配置 `MASTER_LOG_FILE` 

`Position` 对应从库配置 `MASTER_LOG_POS`

## 从服务器

1、从服务器修改配置

```bash
vi /etc/my.cnf
```

修改以下内容

```bash
[mysqld]
server-id = 2
```

2、重启 MySQL 服务

```bash
sudo systemctl restart mysqld
```

3、设置从服务器连接到主服务器

```sql
CHANGE MASTER TO
  MASTER_HOST = 'master_host_ip',  -- 主服务器的 IP 地址
  MASTER_USER = 'replica_user',    -- 主服务器上创建的复制用户
  MASTER_PASSWORD = 'your_password', -- 复制用户的密码
  MASTER_LOG_FILE = 'master_log_file',  -- 主服务器的二进制日志文件
  MASTER_LOG_POS = master_log_pos;    -- 主服务器的二进制日志位置
```

`MASTER_HOST` 是主服务器的 IP 地址。

`MASTER_USER` 和 `MASTER_PASSWORD` 是主服务器上为复制创建的账户和密码。

`MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 是从主服务器获取的二进制日志文件和位置。

4、启动复制进程

```
START SLAVE;
```

5、检查复制状态

```
SHOW SLAVE STATUS\G
```

你应该看到类似下面的输出，特别是 `Slave_IO_Running` 和 `Slave_SQL_Running` 两个字段都应该是 `Yes`，表示复制正常工作。

## 验证主从复制

1、在主服务器上创建或修改数据

```sql
CREATE DATABASE test_db;
USE test_db;
CREATE TABLE test_table (id INT);
INSERT INTO test_table VALUES (1);
```

## 常见问题

1、问题1

```
Coordinator stopped because there were error(s) in the worker(s). The most recent failure being: Worker 1 failed executing transaction 'ANONYMOUS' at source log mysql-bin.000001, end_log_pos 1948. See error log and/or performance_schema.replication_applier_status_by_worker table for more details about this failure or others, if any.
```

**错误位置**：在主库 binlog 文件 `mysql-bin.000001` 的 `end_log_pos = 1948` 处的一个事务执行失败。

**解决方案1**

测试环境可暂时跳过，但如果多次跳过，说明主从数据已经严重不一致，必须重新初始化从库！

```
STOP REPLICA;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START REPLICA;
```

**解决方案2**

切换到新的 binlog 文件

```bash
# 在主库手动刷新 binlog，生成一个新的干净的 binlog 文件：
FLUSH LOGS;
SHOW MASTER STATUS;

# 在从库上重新配置复制，使用新的 binlog 文件和位置
STOP REPLICA;
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='10.91.11.106',
    SOURCE_USER='replica_user',
    SOURCE_PASSWORD='your_password',
    SOURCE_LOG_FILE='mysql-bin.000002',
    SOURCE_LOG_POS=154;
START REPLICA;
```