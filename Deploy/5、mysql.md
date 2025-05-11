# docker 安装

## 1、更改镜像源

创建或修改配置文件

```bash
sudo vi /etc/docker/daemon.json
```

添加国内镜像源地址

```bash
{
  "registry-mirrors": [
    "https://registry.docker-cn.com",          // Docker 中国官方镜像（推荐）
    "https://mirror.ccs.tencentyun.com",       // 腾讯云镜像
    "https://docker.mirrors.ustc.edu.cn",      // 中科大镜像
    "https://hub-mirror.c.163.com",            // 网易云镜像
    "https://<你的ID>.mirror.aliyuncs.com"     // 阿里云镜像（需注册后获取）
  ]
}
```

重启 Docker 服务

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

验证配置是否生效

```bash
docker info

Registry Mirrors:
  https://registry.docker-cn.com/
  https://mirror.ccs.tencentyun.com/
```

## 2、拉取镜像

## 3、启动

# yum 安装

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

# mysql

## 1、数据库初始化

修改密码，配置权限

```sql
# 获取默认 root 密码
sudo grep 'temporary password' /var/log/mysqld.log
# 登录
mysql -uroot -p

ALTER USER 'root'@'localhost' IDENTIFIED BY 'Aa123456..!';

use mysql;

update user set host='%' where user='root';

FLUSH PRIVILEGES;
```

## 2、yum 安装目录

| 类型                         | 路径                                       | 说明                                       |
| ---------------------------- | ------------------------------------------ | ------------------------------------------ |
| 📄 主程序                     | `/usr/sbin/mysqld`                         | MySQL Server 可执行文件                    |
| 📁 配置文件                   | `/etc/my.cnf`                              | MySQL 的主配置文件                         |
| 📁 数据目录（首次启动后创建） | `/var/lib/mysql/`                          | 存放数据库数据、表、用户信息等             |
| 📁 日志目录                   | `/var/log/mysqld.log`                      | MySQL 启动及运行日志（包括初始 root 密码） |
| 📁 系统服务文件               | `/usr/lib/systemd/system/mysqld.service`   | 用于 systemd 启动 MySQL                    |
| 📁 客户端工具                 | `/usr/bin/mysql`、`/usr/bin/mysqladmin` 等 | MySQL 客户端及管理工具                     |
| 📁 库文件                     | `/usr/lib64/mysql/`                        | MySQL 相关动态库                           |
| 📁 通用文件                   | `/usr/share/mysql/`                        | 包括错误信息、字符集、SQL 脚本等           |

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
CREATE USER 'replica_user'@'%' IDENTIFIED BY 'Aa123456..!';
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