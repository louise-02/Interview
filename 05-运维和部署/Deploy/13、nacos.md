# Nacos 简介

[Nacos 官网](https://nacos.io) | [GitHub Releases](https://github.com/alibaba/nacos/releases)

Nacos 是阿里巴巴开源的**注册中心**和**配置中心**，Spring Cloud Alibaba 微服务架构中常用。

| 功能 | 说明 |
| ---- | ---- |
| 服务注册与发现 | 微服务启动后注册到 Nacos，其他服务通过服务名调用 |
| 配置管理 | 集中管理配置文件，支持动态刷新 |
| 命名空间 | 隔离 dev / test / prod 环境 |

常用端口：

| 端口 | 说明 |
| ---- | ---- |
| 8848 | Web 控制台 + HTTP API |
| 9848 | gRPC（Nacos 2.x 客户端必连） |
| 9849 | 集群节点间 gRPC（集群模式） |

默认账号密码：`nacos` / `nacos`（**首次登录后请修改**）。

## 部署方式推荐

| 环境 | 推荐方式 | 说明 |
| ---- | -------- | ---- |
| **生产** | **二进制 + MySQL** | 数据持久化、集群稳定；需 JDK 17+ |
| 开发 / 测试 | Docker（Derby 内嵌库） | 单机快速体验，**不建议生产使用** |

---

# docker 安装（Nacos 2.4.3）

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)  
> 偏生产说明见 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md)

## 1、创建目录

```bash
mkdir -p /data/docker/nacos/{logs,data}
```

## 2、偏生产配置（环境变量 + MySQL）

| 模式 | 用途 | compose |
| ---- | ---- | ------- |
| Derby 单机 | 开发测试 | [nacos.yml](./docker/nacos.yml) |
| **MySQL 持久化** | **偏生产 / Docker 推荐** | [nacos-mysql.yml](./docker/nacos-mysql.yml) |

偏生产务必用 **MySQL 模式**，修改 [nacos-mysql.yml](./docker/nacos-mysql.yml) 中 `MYSQL_SERVICE_*`、`JVM_XMX`，并先在 MySQL 中建库导入 `mysql-schema.sql`（见下方 §3.1）。初始化、命名空间、备份见下文 [Nacos 配置与运维](#nacos-配置与运维)。

## 3、单机模式（Derby，开发测试）

```bash
cd /path/to/Deploy/docker
docker compose -f nacos.yml up -d
```

## 4、单机模式（MySQL，偏生产推荐）

### 4.1、初始化 MySQL 数据库

```sql
CREATE DATABASE nacos_config CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

CREATE USER 'nacos'@'%' IDENTIFIED BY 'nacos123';
GRANT ALL PRIVILEGES ON nacos_config.* TO 'nacos'@'%';
FLUSH PRIVILEGES;
```

导入表结构（从 Nacos 安装包或 GitHub 获取 `mysql-schema.sql`）：

```bash
# 方式1：从安装包获取
tar -zxvf nacos-server-2.4.3.tar.gz
mysql -unacos -p nacos_config < nacos/conf/mysql-schema.sql

# 方式2：从 GitHub 下载
wget https://raw.githubusercontent.com/alibaba/nacos/2.4.3/distribution/conf/mysql-schema.sql
mysql -unacos -p nacos_config < mysql-schema.sql
```

### 4.2、启动

修改 [docker/nacos-mysql.yml](./docker/nacos-mysql.yml) 中 `MYSQL_SERVICE_HOST` 为实际 MySQL 地址，然后：

```bash
cd /path/to/Deploy/docker
docker compose -f nacos-mysql.yml up -d
```

## 5、验证

```bash
docker logs -f nacos
# 看到 Nacos started successfully in standalone mode 表示启动成功
```

浏览器访问 `http://宿主机IP:8848/nacos`，登录 `nacos/nacos`。

## 6、Spring Cloud 接入

`application.yml` 示例：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 宿主机IP:8848
      config:
        server-addr: 宿主机IP:8848
        file-extension: yaml
```

Maven 依赖见 [Maven.md](../../02-技术栈/Maven/Maven.md) 中 Spring Cloud Alibaba 部分。

## 7、常用操作

```bash
docker compose -f docker/nacos.yml ps
docker compose -f docker/nacos.yml logs -f
docker compose -f docker/nacos.yml restart
docker compose -f docker/nacos.yml down
```

# 二进制包安装（Nacos 2.4.3）

Nacos 2.x 需要 **JDK 17+**，参考 [2、jdk.md](./2、jdk.md)。

## 1、用户和目录创建

```bash
sudo useradd -r -s /sbin/nologin nacos 2>/dev/null || true

sudo mkdir -p /opt/nacos
sudo mkdir -p /data/nacos/2.4.3/{logs,data}

sudo chown -R nacos:nacos /opt/nacos
sudo chown -R nacos:nacos /data/nacos
```

## 2、下载安装

```bash
cd /opt/src
wget https://github.com/alibaba/nacos/releases/download/2.4.3/nacos-server-2.4.3.tar.gz
sudo tar -zxvf nacos-server-2.4.3.tar.gz -C /opt/nacos --strip-components=1
sudo chown -R nacos:nacos /opt/nacos

# 版本软链接
sudo ln -sfn /opt/nacos /opt/nacos/current
sudo ln -sfn /data/nacos/2.4.3 /data/nacos/current
```

## 3、单机模式（Derby，开发测试）

```bash
cd /opt/nacos/current/bin
sudo -u nacos sh startup.sh -m standalone

# 查看日志
tail -f /opt/nacos/current/logs/start.out

# 停止
sudo -u nacos sh shutdown.sh
```

## 4、单机模式（MySQL，生产推荐）

### 4.1、初始化数据库

同 Docker 版 3.1 节，创建 `nacos_config` 库并导入 `mysql-schema.sql`。

### 4.2、修改配置

```bash
vi /opt/nacos/current/conf/application.properties
```

```properties
# 使用 MySQL 持久化
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false&allowPublicKeyRetrieval=true
db.user.0=nacos
db.password.0=nacos123

# 单机模式
nacos.standalone=true
```

### 4.3、启动

```bash
cd /opt/nacos/current/bin
sudo -u nacos sh startup.sh -m standalone
```

## 5、集群模式（简要）

集群至少 3 个节点，每个节点配置 `cluster.conf`：

```bash
vi /opt/nacos/current/conf/cluster.conf
```

```
192.168.1.101:8848
192.168.1.102:8848
192.168.1.103:8848
```

每个节点修改 `application.properties`：

```properties
nacos.inetutils.ip-address=本机IP
```

集群必须使用 **MySQL** 持久化，不能使用 Derby。启动时不加 `-m standalone`：

```bash
sudo -u nacos sh startup.sh
```

## 6、注册 systemd 服务

```bash
sudo tee /etc/systemd/system/nacos.service > /dev/null <<'EOF'
[Unit]
Description=Nacos Server
After=network.target

[Service]
Type=forking
User=nacos
Group=nacos
Environment="JAVA_HOME=/opt/jdk/jdk17"
ExecStart=/opt/nacos/current/bin/startup.sh -m standalone
ExecStop=/opt/nacos/current/bin/shutdown.sh
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable nacos
sudo systemctl start nacos
sudo systemctl status nacos
```

## 7、环境变量

```bash
vi /etc/profile.d/nacos.sh

export NACOS_HOME=/opt/nacos/current
export PATH=$PATH:$NACOS_HOME/bin

source /etc/profile.d/nacos.sh
```

# Nacos 配置与运维

## 1、二进制安装目录

| 类型 | 路径 | 说明 |
| ---- | ---- | ---- |
| 主程序 | `/opt/nacos/current/bin/startup.sh` | 启动脚本 |
| 配置文件 | `/opt/nacos/current/conf/application.properties` | 主配置 |
| 集群配置 | `/opt/nacos/current/conf/cluster.conf` | 集群节点列表 |
| SQL 脚本 | `/opt/nacos/current/conf/mysql-schema.sql` | MySQL 建表脚本 |
| 日志目录 | `/opt/nacos/current/logs/` | 运行日志 |
| 数据目录 | `/data/nacos/current/data/` | 自定义数据目录（Derby 模式） |

## 2、Docker 安装目录

| 类型 | 宿主机路径 | 容器路径 |
| ---- | ---------- | -------- |
| 日志 | `/data/docker/nacos/logs` | `/home/nacos/logs` |
| 数据 | `/data/docker/nacos/data` | `/home/nacos/data` |

## 3、命名空间与配置管理

登录控制台 → **命名空间** → **新建命名空间**：

| 命名空间 ID | 名称 | 用途 |
| ----------- | ---- | ---- |
| dev | 开发环境 | 本地开发 |
| test | 测试环境 | 测试联调 |
| prod | 生产环境 | 线上 |

在 **配置管理** 中创建 Data ID（如 `application.yaml`），Group 默认 `DEFAULT_GROUP`。

客户端通过 `spring.cloud.nacos.config.namespace` 指定命名空间 ID。

## 4、开放防火墙

```bash
sudo firewall-cmd --permanent --add-port=8848/tcp
sudo firewall-cmd --permanent --add-port=9848/tcp
sudo firewall-cmd --reload
```

## 5、初始化与备份

```bash
# 1. 修改默认 nacos/nacos 密码
# 控制台 → 权限控制 → 用户列表

# 2. 创建 prod 命名空间，迁移配置
# 控制台 → 命名空间 → 新建 prod

# 3. 备份 MySQL 中的 nacos 库
mysqldump -u root -p nacos > /backup/nacos_$(date +%F).sql

# 4. 健康检查
curl http://127.0.0.1:8848/nacos/v1/console/health/readiness

# 5. 集群模式检查各节点 logs/nacos.log
tail -f /data/nacos/current/logs/nacos.log
```

## 6、常见问题

**启动失败：Unable to find Java**

确认 `JAVA_HOME` 指向 JDK 17+：

```bash
echo $JAVA_HOME
java -version
```

**客户端连不上，控制台正常**

Nacos 2.x 客户端除了 8848，还需要 **9848** 端口可达。

**Docker 启动后 derby 数据丢失**

Derby 模式需挂载 `/data/docker/nacos/data`；生产环境请改用 MySQL。

**修改默认密码**

控制台 → **权限控制** → **用户列表** → 编辑 `nacos` 用户密码。
