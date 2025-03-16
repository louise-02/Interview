# 1、添加官方仓库

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
```

# 2、安装 ClickHouse 服务器和客户端

```bash
sudo yum install -y clickhouse-server clickhouse-client
```

# 3、启动 ClickHouse 服务器

```bash
sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
sudo systemctl status clickhouse-server
clickhouse-client # 或 "clickhouse-client --password" 如果您设置了密码。
```

# 4、配置远程访问

默认仅允许本地连接。若需远程访问，修改配置文件：

```bash
sudo vim /etc/clickhouse-server/config.xml
```

找到 `<listen_host>` 配置项，取消注释并修改为：

```xml
<listen_host>0.0.0.0</listen_host>
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

# 5、文件路径

**数据存储**：`/var/lib/clickhouse/data/`

**日志文件**：`/var/log/clickhouse-server/clickhouse-server.log`

**主配置文件**：`/etc/clickhouse-server/config.xml`

**用户权限配置**：`/etc/clickhouse-server/users.xml`

# 6、卸载 ClickHouse

```bash
sudo systemctl stop clickhouse-server
sudo yum remove clickhouse-server clickhouse-client
sudo rm -rf /var/lib/clickhouse /etc/clickhouse-server /etc/clickhouse-client
```

