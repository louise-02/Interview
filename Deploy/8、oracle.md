# yum 安装（未验证成功）

## 1、安装

**1、安装必要依赖（Oracle 官方推荐）**

安装这个包会自动配置系统内核参数、用户、组等。

```bash
# 配置官方源
sudo wget https://yum.oracle.com/repo/OracleLinux/OL7/latest/x86_64/getPackage/oraclelinux-release-el7-1.0-10.el7.x86_64.rpm
sudo rpm -ivh oraclelinux-release-el7-1.0-10.el7.x86_64.rpm

# Oracle GPG 密钥配置
sudo wget https://yum.oracle.com/RPM-GPG-KEY-oracle-ol7 -O /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle

# 清理 yum 缓存
sudo yum clean all

# 下载19c配置
sudo yum install -y oracle-database-preinstall-19c
```

**2、安装 Oracle 数据库 RPM 包**

注册并登录 Oracle 官方网站（需Oracle账号），下载 Oracle Database 19c for Linux x86-64 RRM 包。

https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html

```bash
sudo yum -y localinstall oracle-database-ee-19c-1.0-1.x86_64.rpm
```

**3、配置数据库实例（初始化）**

RPM 安装不会自动创建数据库，需要运行初始化脚本

```bash
sudo /etc/init.d/oracledb_ORCLCDB-19c configure
```

> 该命令会自动：
>
> - 创建 ORCLCDB 数据库（CDB）
>
> - 设置字符集为 AL32UTF8
>
> - 设置默认密码（在 `/etc/sysconfig/oracledb_ORCLCDB-19c.conf` 中定义）
>
> - 启动监听器和数据库

**4、启停数据库**

```bash
# 启动
sudo systemctl start oracle-database

# 停止
sudo systemctl stop oracle-database
```

**5、监听器启停**

yum 安装不会配置 listener 的 systemctl

```bash
# 启动监听器
lsnrctl start

# 停止监听器
lsnrctl stop

# 重载监听器
lsnrctl reload
```

## 2、卸载



# 二进制包安装

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
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmax = 8589934592       # 根据内存大小调整，比如 8 GB
kernel.shmall = 2097152
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



# oracle

## 1、yum 安装目录

| 类型               | 路径                                             | 说明                                        |
| ------------------ | ------------------------------------------------ | ------------------------------------------- |
| 📁 软件安装目录     | `/opt/oracle/product/19c/dbhome_1`               | Oracle 主目录                               |
| 📁 数据文件路径     | `/opt/oracle/oradata/ORCLCDB`                    | 数据库文件                                  |
| 📄 数据库配置文件   | `/etc/sysconfig/oracledb_ORCLCDB-19c.conf`       | 初始化配置，包含密码、字符集等              |
| 📄 启动脚本         | `/etc/init.d/oracledb_ORCLCDB-19c`               | 用于手动启动/停止数据库的脚本               |
| 📄 oratab 文件      | `/etc/oratab`                                    | 控制实例是否随系统启动                      |
| 📁 监听配置         | `/opt/oracle/product/19c/dbhome_1/network/admin` | listener.ora、tnsnames.ora 所在目录         |
| 📁 日志目录         | `/opt/oracle/diag/`                              | 各类日志文件，包括监听器和数据库告警日志    |
| 🧪 环境变量建议文件 | `/home/oracle/.bash_profile`                     | 设置 `$ORACLE_HOME`、`$ORACLE_SID`、`$PATH` |

## 3、创建授权用户

以 VLMP_USER 用户和 VLMP 表空间为例。

创建用户

```sql
CREATE USER VLMP_USER IDENTIFIED BY password;
```

分配表空间

```bash
ALTER USER VLMP_USER DEFAULT TABLESPACE VLMP;
```

授权

```sql
GRANT 
  CREATE SESSION,         -- 连接数据库
  CREATE TABLE,           -- 创建表
  CREATE VIEW,            -- 创建视图
  CREATE SEQUENCE,        -- 创建序列
  CREATE PROCEDURE,       -- 创建存储过程/函数
  CREATE TRIGGER,         -- 创建触发器
  CREATE TYPE,            -- 创建用户定义类型
  CREATE SYNONYM          -- 创建同义词
TO VLMP_USER;
```

授权对此表空间的所有权限

```sql
ALTER USER VLMP_USER QUOTA UNLIMITED ON VLMP; -- 用户在表空间 VLMP 上没有存储限制
```

> **QUOTA**：表示用户在特定表空间上可以使用多少存储空间。
>
> **UNLIMITED**：表示该用户在 `VLMP` 表空间中没有存储空间限制，即可以使用该表空间的所有可用空间。
>
> **ON VLMP**：表示这个设置是针对表空间 `VLMP` 的。