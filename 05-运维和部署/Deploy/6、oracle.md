# Oracle Database 简介

[Oracle 官网](https://www.oracle.com/database/) | [19c 下载](https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html)

Oracle Database 是商业**关系型数据库**，常用于企业核心系统、金融、政务等对稳定性与功能要求较高的场景。

| 特点 | 说明 |
| ---- | ---- |
| 企业级 | 表空间、RAC、Data Guard 等高可用方案 |
| 授权 | 需 Oracle 账号下载安装包，注意许可证 |
| 资源占用 | 对内存、磁盘、内核参数有较高要求 |

常用端口：

| 端口 | 说明 |
| ---- | ---- |
| 1521 | 监听器，客户端连接 |

## 部署方式推荐

| 环境 | 推荐方式 | 说明 |
| ---- | -------- | ---- |
| **生产** | **二进制（唯一）** | 官方静默安装，无 Docker 官方镜像；授权与运维复杂，建议专用 DBA |

---

# 二进制包安装（Oracle 19c）

## 0、磁盘查看

查看磁盘信息

```bash
fdisk -l

磁盘 /dev/sda：107.4 GB, 107374182400 字节，209715200 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x000148e1

   设备 Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048   209715166   104856559+  83  Linux

磁盘 /dev/sdb：429.5 GB, 429496729600 字节，838860800 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
```

若此时发现 /dev/sdb 不在 df -h 中，则需要对硬盘进行格式化

```
# 格式化磁盘为xfs（推荐）
mkfs.xfs /dev/sdb

# 检查文件系统
blkid /dev/sdb
```

挂载磁盘

```
mkdir -p /u02
mount /dev/sdb /u02
```

## 1、安装

**1、安装依赖包**

```bash
# 以 Oracle Linux 7 为例
yum install -y binutils \
               gcc \
               gcc-c++ \
               glibc \
               glibc-devel \
               libaio \
               libaio-devel \
               libX11 \
               libXau \
               libXi \
               libXtst \
               libXrender \
               make \
               sysstat \
               unixODBC \
               unixODBC-devel

yum install -y binutils gcc gcc-c++ glibc glibc-devel libaio libaio-devel \
               libX11 libXau libXi libXtst libXrender make sysstat \
               unixODBC unixODBC-devel compat-libstdc++-33 compat-libcap1 \
               ksh smartmontools glibc-headers libicu libicu-devel \
               libnsl oracle-database-preinstall-19c xorg-x11-utils perl \
               libpcap
```

**2、创建用户与组**

```bash
# 创建操作系统组
groupadd oinstall
groupadd dba
groupadd oper

# 创建 oracle 用户，设置组并创建家目录
useradd -g oinstall -G dba,oper -d /home/oracle -m oracle
passwd oracle
```

**3、配置内核参数与用户资源限制**

编辑 `/etc/sysctl.conf`，追加：

```bash
# 最大文件句柄数（保持默认或稍低）
fs.file-max = 6815744
# Semaphore（信号量）
kernel.sem = 250 32000 100 128

# 共享内存上限（须 ≥ 后续 SGA，且不超过物理内存约 50%～60%）
# 16GB 机器示例 8G：
kernel.shmmax = 8589934592
# shmmax / 4096（4KB 页大小）
kernel.shmall = 2097152

# 网络缓冲区
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
fs.aio-max-nr = 1048576
```

> kernel.sem 可以考虑改为 32000 1024000000 500 128

执行 `sysctl -p` 使生效。

编辑 `/etc/security/limits.conf`，追加：

```bash
oracle   soft   nproc    2047
oracle   hard   nproc    16384
oracle   soft   nofile   65536
oracle   hard   nofile   65536
oracle   soft   stack    10240
oracle   soft   memlock  unlimited
oracle   hard   memlock  unlimited
```

设置 shell 环境（在 `/home/oracle/.bash_profile` 或 `/home/oracle/.bashrc` 中）：

```bash
# Oracle 环境变量
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.0.0/dbhome_1
export ORACLE_SID=ORCLCDB
export PATH=$ORACLE_HOME/bin:$PATH
```

刷新环境变量

```bash
source /home/oracle/.bash_profile
```

**4、准备安装目录**

```bash
# 切换到 root
mkdir -p /u01/app/oracle
mkdir -p /u01/app/oraInventory
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
mkdir -p /u02
chown -R oracle:oinstall /u01 /u02
chmod -R 775 /u01 /u02
```

**5、获取并解压 Oracle 安装包**

注册并登录 Oracle 官方网站（需Oracle账号），下载 Oracle Database 19c for Linux x86-64。

https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html

将安装包上传到服务器，例如放在 /home/oracle 下。

```bash
su - oracle
cd /home/oracle
unzip LINUX.X64_193000_db_home.zip -d $ORACLE_HOME
```

**6、启动安装程序，静默安装（Silent Mode）**

切换至 oracle 用户，创建响应文件

```bash
su - oracle
cp $ORACLE_HOME/install/response/db_install.rsp /home/oracle/db_install.rsp
vi /home/oracle/db_install.rsp
```

```bash
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v19.0.0
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=your-hostname
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
SELECTED_LANGUAGES=en,zh_CN

ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.OSDBA_GROUP=dba
oracle.install.db.OSOPER_GROUP=oper
oracle.install.db.OSBACKUPDBA_GROUP=dba
oracle.install.db.OSDGDBA_GROUP=dba
oracle.install.db.OSKMDBA_GROUP=dba
oracle.install.db.OSRACDBA_GROUP=dba

oracle.install.db.rootconfig.executeRootScript=false
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
DECLINE_SECURITY_UPDATES=true
```

> ORACLE_HOSTNAME 要写主机名，可用 hostname 命令查看
>
> 修改主机名：sudo hostnamectl set-hostname 新主机名

执行静默安装

```bash
cd $ORACLE_HOME
./runInstaller -silent -responseFile /home/oracle/db_install.rsp \
    -ignorePrereq -waitforcompletion
```

安装完成后，以 root 身份执行脚本

```bash
su - root
/u01/app/oraInventory/orainstRoot.sh
/u01/app/oracle/product/19.0.0/dbhome_1/root.sh
```

**7、配置监听器（Listener）**

切换至 oracle 用户，编辑或创建 `$ORACLE_HOME/network/admin/listener.ora`

```bash
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = your.server.hostname/ip)(PORT = 1521))
    )
  )
```

启动监听

```bash
lsnrctl start
lsnrctl status
```

**8、使用 DBCA 创建数据库**

安装完软件后，可以用 Database Configuration Assistant 创建数据库。

```bash
# 静默模式示例
su - oracle 
dbca -silent \
     -createDatabase \
     -templateName General_Purpose.dbc \
     -gdbName ORCLCDB \
     -sid ORCLCDB \
     -responseFile NO_VALUE \
     -characterSet AL32UTF8 \
     -sysPassword SYS_password \
     -systemPassword SYSTEM_password \
     -storageType FS \
     -datafileDestination /u02/oradata \
     -redoLogFileSize 50 \
     -emConfiguration LOCAL
```

> `gdbName`：全局数据库名（Global DB Name）
>
> `sid`：实例名
>
> `datafileDestination`：数据文件存放路径
>
> `sysPassword`：sys 用户的密码
>
> `systemPassword`：system 用户的密码
>
> `totalMemory`：24576 总内存24G

验证监听器是否注册数据库服务

```bash
lsnrctl status
# 应该看到以下内容
Service "ORCLCDB" has 1 instance(s).
```

如果没有自动注册，再手动注册

```bash
su - oracle
# 设置 NLS_LANG 环境变量匹配 UTF-8
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
sqlplus / as sysdba

# 设置注册地址
ALTER SYSTEM SET local_listener='(ADDRESS = (PROTOCOL=TCP)(HOST=your.server.hostname/ip)(PORT=1521))' SCOPE=BOTH;
show parameter local_listener;

alter system register;
```

**9、验证与日常运维**

登录数据库

```bash
su - oracle
sqlplus / as sysdba
SQL> SELECT name, open_mode FROM v$database;
```

检查表空间、状态

```sql
SELECT tablespace_name, status FROM dba_tablespaces;
```

**10、navicate 连接**

服务名：ORCLCDB

端口：1521

用户名 sys 密码 SYS_password  角色 SYSDBA

用户名 system 密码 SYSTEM_password

**11、启动与关闭**

关闭

```bash
su - oracle
lsnrctl stop
sqlplus / as sysdba
> shutdown immediate;
> exit
```

启动

```bash
su - oracle
lsnrctl start
sqlplus / as sysdba
> startup;
# 查看是否多租户
SELECT CDB FROM V$DATABASE;
# 如果你使用的是多租户（CDB），并且还需要启动 PDB（可插拔数据库）
ALTER PLUGGABLE DATABASE ALL OPEN;
> exit
```

**12、设置开机自启**

切换至 root 用户

创建监听器 systemd 服务

```bash
sudo tee /etc/systemd/system/oracle-listener.service > /dev/null <<EOF
[Unit]
Description=Oracle Listener
After=network.target

[Service]
Type=forking
User=oracle
Group=oinstall
Environment="ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1"
ExecStart=/u01/app/oracle/product/19.0.0/dbhome_1/bin/lsnrctl start
ExecStop=/u01/app/oracle/product/19.0.0/dbhome_1/bin/lsnrctl stop
ExecReload=/u01/app/oracle/product/19.0.0/dbhome_1/bin/lsnrctl reload
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

创建数据库 systemd 服务

```bash
sudo tee /etc/systemd/system/oracle-database.service > /dev/null <<EOF
[Unit]
Description=Oracle Database Service
After=oracle-listener.service

[Service]
Type=forking
User=oracle
Group=oinstall
Environment="ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1"
Environment="ORACLE_SID=ORCLCDB"
ExecStart=/u01/app/oracle/product/19.0.0/dbhome_1/bin/dbstart $ORACLE_HOME
ExecStop=/u01/app/oracle/product/19.0.0/dbhome_1/bin/dbshut $ORACLE_HOME
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

配置 oratab

```bash
vi /etc/oratab
# 末尾的 Y 表示允许 dbstart 自动启动此实例
ORCLCDB:/u01/app/oracle/product/19.0.0/dbhome_1:Y
```

启用并启动服务

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload

sudo systemctl enable oracle-listener.service
sudo systemctl enable oracle-database.service

sudo systemctl start oracle-listener.service
sudo systemctl start oracle-database.service
```

验证状态

```bash
sudo systemctl status oracle-listener.service
sudo systemctl status oracle-database.service
```

如果启动服务遇到时间比较长的情况，服务需要添加，单位秒

```bash
[Service]
TimeoutStartSec=600
```

## 2、卸载

停止所有 Oracle 实例与监听器

```bash
su - oracle
lsnrctl stop 或 systemctl stop oracle-listener.service
sqlplus / as sysdba
> shutdown immediate;
> exit
```

删除 Oracle 服务（如果使用了 systemd 自定义服务）

```bash
su root
sudo systemctl stop oracle-listener.service
sudo systemctl stop oracle-database.service

sudo systemctl disable oracle-listener.service
sudo systemctl disable oracle-database.service

sudo rm -f /etc/systemd/system/oracle-listener.service
sudo rm -f /etc/systemd/system/oracle-database.service
sudo systemctl daemon-reexec
```

删除安装目录与相关文件

```bash
sudo rm -rf /u01/app/oracle
sudo rm -rf /u01/app/oraInventory
sudo rm -rf /u02/oradata
```

删除 Oracle 用户与用户组

```bash
# 查看 oracle 用户有哪些正在运行的进程
ps -u oracle
# 杀掉所有 oracle 用户的进程 也可以 kill -9 <pid> 来删除
sudo pkill -u oracle

sudo userdel -r oracle
sudo groupdel oinstall
sudo groupdel dba
```

清理环境变量

```bash
sudo vi /home/oracle/.bash_profile 或 sudo vi /etc/profile

# 删除以下内容
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.0.0/dbhome_1
export ORACLE_SID=ORCLCDB
export PATH=$ORACLE_HOME/bin:$PATH

source /home/oracle/.bash_profile 或 source /etc/profile
```

清理 oratab 文件

```bash
sudo rm -f /etc/oratab
```

## 3、常见问题

1、dbca 安装数据库时提示 SGA 大小问题

```bash
[WARNING] [DBT-11207] Specified SGA size is greater than the shmmax on the system. The database creation might fail with "ORA-27125 - Unable to create shared memory segment error".
   ACTION: Specify SGA size lesser than or equal to the shmmax on the system.
Prepare for db operation
10% complete
Copying database files
12% complete
[WARNING] ORA-27104: system-defined limits for shared memory was misconfigured

[FATAL] ORA-01034: ORACLE not available

40% complete
100% complete
[FATAL] ORA-01034: ORACLE not available

10% complete
0% complete
Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/ORCLCDB/ORCLCDB5.log" for further details.
```

root 用户修改配置

如果没有指定 SGA_TARGET 或 SGA_MAX_SIZE，默认情况下 Oracle 会自动分配内存给 SGA，通常是服务器总内存的 30% 到 50% 左右。

```bash
vi /etc/sysctl.conf

# 如服务器为32G内存 改为16G
kernel.shmmax = 17179869184
kernel.shmall = 4194304

# 使配置生效
sudo sysctl -p
```

## 4、Swap

查看当前是否存在 swap

```bash
# 查看 swap 使用情况
swapon --show
# 或
free -h

如果输出为空，说明 当前没有 swap。
```

创建 swap 文件

```bash
# 1. 创建一个16GB的swap文件
sudo fallocate -l 16G /swapfile

# 如果 fallocate 不可用，可用 dd（速度慢但通用）
# sudo dd if=/dev/zero of=/swapfile bs=1G count=16 status=progress

# 2. 设置权限
sudo chmod 600 /swapfile

# 3. 格式化为 swap
sudo mkswap /swapfile

# 4. 启用 swap
sudo swapon /swapfile

# 5. 验证是否启用成功
swapon --show
free -h
```

设置开机自动挂载

```bash
否则重启后 swap 会失效。

sudo vim /etc/fstab

在文件末尾加上
/swapfile none swap sw 0 0
```

删除 swap

```bash
sudo swapoff /swapfile
sudo rm -f /swapfile
sudo sed -i '/\/swapfile/d' /etc/fstab
```

# Oracle 配置与运维

## 1、安装目录

| 类型                   | 路径                                                     | 说明                                           | 来源                        |
| ---------------------- | -------------------------------------------------------- | ---------------------------------------------- | --------------------------- |
| 📁 **基础目录**         | `/u01/app/oracle`                                        | Oracle 基础目录，包含各种子目录                | `ORACLE_BASE` 参数          |
| 📁 **软件安装目录**     | `/u01/app/oracle/product/19.0.0/dbhome_1`                | Oracle 软件主目录，包含所有二进制文件和工具    | `ORACLE_HOME`` 参数         |
| 📁 **清单位录**         | `/u01/app/oraInventory`                                  | Oracle 产品清单，记录所有安装的 Oracle 产品    | `INVENTORY_LOCATION` 参数   |
| 📁 **数据文件目录**     | `/u02/oradata`                                           | 数据库数据文件、控制文件、重做日志文件存储位置 | `-datafileDestination` 参数 |
| 📁 **数据库文件目录**   | `/u02/oradata/ORCLCDB/`                                  | 具体数据库文件，如 system01.dbf                | DBCA 自动创建               |
| 📁 **网络配置目录**     | `/u01/app/oracle/product/19.0.0/dbhome_1/network/admin/` | 网络配置文件，如 listener.ora                  | `ORACLE_HOME` 下的固定路径  |
| 📁 **诊断日志目录**     | `/u01/app/oracle/diag/`                                  | ADR 诊断日志目录，包含跟踪文件、告警日志等     | 自动创建                    |
| 🧪 **环境变量建议文件** | `/home/oracle/.bash_profile`                             | 设置 `$ORACLE_HOME`、`$ORACLE_SID`、`$PATH`    |                             |

## 2、创建表空间

创建 VMLP 表空间

```bash
CREATE TABLESPACE VLMP
DATAFILE '/u02/oradata/ORCLCDB/vlmp01.dbf' SIZE 20G
AUTOEXTEND ON NEXT 1G MAXSIZE 200G
EXTENT MANAGEMENT LOCAL
SEGMENT SPACE MANAGEMENT AUTO
LOGGING
ONLINE;
```

> 文件存储在 /u02/oradata/ORCLCDB/vlmp01.dbf 初始化 20G
>
> 每次增加 1G 最大 200G

```bash
CREATE TABLESPACE VLMP
DATAFILE '/u02/oradata/ORCLCDB/vlmp01.dbf' SIZE 20G
AUTOEXTEND ON NEXT 1G MAXSIZE UNLIMITED
EXTENT MANAGEMENT LOCAL
SEGMENT SPACE MANAGEMENT AUTO
LOGGING
ONLINE;
```

> 每次增加 1G 最大无限制

## 3、创建授权用户

以 VLMP_USER 用户和 VLMP 表空间为例。

创建用户

```sql
CREATE USER VLMP IDENTIFIED BY VLMP_Aa123456;
```

分配表空间

```bash
ALTER USER VLMP DEFAULT TABLESPACE VLMP;
```

授权

```sql
GRANT CREATE SESSION,CREATE TABLE,CREATE VIEW,CREATE SEQUENCE,CREATE PROCEDURE,CREATE TRIGGER,CREATE TYPE,CREATE SYNONYM TO VLMP;

# 授权所有权限
GRANT ALL PRIVILEGES TO VLMP;
```

>  CREATE SESSION,         -- 连接数据库
>  CREATE TABLE,           -- 创建表
>  CREATE VIEW,            -- 创建视图
>  CREATE SEQUENCE,        -- 创建序列
>  CREATE PROCEDURE,       -- 创建存储过程/函数
>  CREATE TRIGGER,         -- 创建触发器
>  CREATE TYPE,            -- 创建用户定义类型
>  CREATE SYNONYM          -- 创建同义词

授权对此表空间的所有权限

```sql
ALTER USER VLMP QUOTA UNLIMITED ON VLMP; -- 用户在表空间 VLMP 上没有存储限制
```

> **QUOTA**：表示用户在特定表空间上可以使用多少存储空间。
>
> **UNLIMITED**：表示该用户在 `VLMP` 表空间中没有存储空间限制，即可以使用该表空间的所有可用空间。
>
> **ON VLMP**：表示这个设置是针对表空间 `VLMP` 的。

## 4、系统参数

Oracle 对共享内存、信号量、文件描述符要求较高，安装阶段见上文「安装前准备 → 内核参数与资源限制」。

| 项 | 建议 | 说明 |
| -- | ---- | ---- |
| `kernel.shmmax` | ≥ SGA 大小，且 **≤ 物理内存的 50%～60%** | 小于 SGA 会报 ORA-27125 |
| `kernel.shmall` | `shmmax / 4096`（4KB 页） | 与 shmmax 配套 |
| SGA（`sga_target`） | 专用库 **40%～60%** 物理内存 | 混部酌减；PGA 另计 |
| Swap | 建议 **8GB～16GB** 或物理内存 50%～100% | 安装文档 §4 有示例 |

16GB 机器示例：`shmmax = 8589934592`（8GB）；32GB 机器：`shmmax = 17179869184`（16GB）。

> `sysctl.conf` 中行尾不要写 `#` 注释，注释需单独成行，否则参数可能解析失败。

## 5、日常启停

```bash
lsnrctl start | stop | status
sqlplus / as sysdba
# SHUTDOWN IMMEDIATE; / STARTUP;
```

## 6、备份

```bash
# RMAN 全量备份（按 DBA 规范调整）
rman target /
BACKUP DATABASE PLUS ARCHIVELOG;
```

## 7、巡检

```sql
SELECT tablespace_name, ROUND(used_percent, 2) FROM dba_tablespace_usage_metrics;
```

## 8、安全

- 应用使用独立账号，最小权限
- 1521 仅对应用网段开放
- 补丁在测试环境验证后再上生产