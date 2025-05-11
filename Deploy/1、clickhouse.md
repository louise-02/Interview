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



