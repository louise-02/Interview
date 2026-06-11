# Centos yum安装

## 1、安装前准备

**系统更新**

```bash
sudo yum update -y
```

**配置主机名和 Hosts 文件（重要，尤其为未来集群准备）**

```bash
# 设置一个有意义的主机名，例如 rabbitmq-node01
sudo hostnamectl set-hostname rabbitmq-node01

# 编辑 /etc/hosts，确保主机名能正确解析到本机IP（127.0.0.1 和 实际IP）
echo "$(hostname -I | awk '{print $1}') $(hostname)" | sudo tee -a /etc/hosts
# 例如：192.168.1.100 rabbitmq-node01
```

**开放防火墙端口**

```bash
sudo firewall-cmd --permanent --add-port=5672/tcp  # AMQP 协议端口，应用程序连接
sudo firewall-cmd --permanent --add-port=15672/tcp # Web 管理界面端口
sudo firewall-cmd --permanent --add-port=25672/tcp # Erlang 分布式节点间通信（集群用）
sudo firewall-cmd --permanent --add-port=4369/tcp  # EPMD (Erlang Port Mapper Daemon) 端口（集群用）
sudo firewall-cmd --reload
```

## 2、安装 Erlang

**下载并安装 Erlang RPM 包**

```bash
# 1. 导入签名密钥
rpm --import https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc
# 如果服务器访问不了 下载到本地上传
rpm --import rabbitmq-release-signing-key.asc

# 2. 添加 Erlang RPM 仓库
# 对于 CentOS 8:
sudo yum install -y https://github.com/rabbitmq/erlang-rpm/releases/download/v23.3.4.11/erlang-23.3.4.11-1.el7.x86_64.rpm
# 对于 CentOS 7，请从上述 GitHub 页面找到对应的仓库包链接
sudo yum install -y erlang-23.3.4.11-1.el7.x86_64.rpm
```

**验证 Erlang 安装**

```bash
erl -version
```

## 3、安装 RabbitMQ Server

**添加 RabbitMQ Yum 仓库**

```bash
# 下载 rabbitmq_server RPM 包（作为仓库源）
# 对于 CentOS 8:
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.28/rabbitmq-server-3.8.28-1.el7.noarch.rpm
sudo yum install -y rabbitmq-server-3.8.28-1.el7.noarch.rpm
```

**安装 RabbitMQ**

```bash
sudo yum install -y rabbitmq-server
```

## 4、配置与管理

**启动服务并设置开机自启**

```bash
sudo systemctl enable rabbitmq-server.service # 启用开机自启
sudo systemctl start rabbitmq-server.service  # 启动服务
sudo systemctl status rabbitmq-server.service # 检查状态，确保是 active (running)
```

**启用 Web 管理插件**

```bash
sudo rabbitmq-plugins enable rabbitmq_management
systemctl restart rabbitmq-server
```

**创建管理员用户（至关重要！）**

默认的 `guest` 用户只能从 `localhost` 访问，**必须创建新用户用于远程管理和应用连接**。

```bash
# 创建用户，用户名 'prod_admin'，密码 'YourStrongPasswordHere!'
sudo rabbitmqctl add_user prod_admin YourStrongPasswordHere!

# 为用户设置 administrator 标签（赋予所有权限）
sudo rabbitmqctl set_user_tags prod_admin administrator

# 授予用户对所有虚拟主机 (vhost) 的配置、写、读权限
sudo rabbitmqctl set_permissions -p / prod_admin ".*" ".*" ".*"

# 修改密码
sudo rabbitmqctl change_password prod_admin 'S3cure!P@ssw0rd2025'
```

**创建专用应用程序用户**

```bash
# 创建一个仅用于连接和发送/消费消息的用户
sudo rabbitmqctl add_user VLMP VLMP_Aa123456
sudo rabbitmqctl set_permissions -p / VLMP ".*" ".*" ".*"
```

## 5、生产环境优化与配置

**配置文件**

RabbitMQ 的主要配置文件位于 `/etc/rabbitmq/rabbitmq.conf`。默认可能不存在，需要手动创建。

```bash
sudo vim /etc/rabbitmq/rabbitmq.conf
```

添加一些基本的生产配置

```bash
# 限制磁盘空闲空间低于 2GB 时触发警报并阻止生产者
disk_free_limit.absolute = 2GB
# 或者使用相对内存大小： disk_free_limit.relative = 2.0

# 禁用 guest 用户，即使本地访问也不行（加强安全）
loopback_users.guest = false

# 配置默认心跳时间（秒），建议与客户端设置一致
heartbeat = 60

# 设置最大连接数和小数点精度
# channel_max = 2047
# frame_max = 131072
```

更高级的配置（如LDAP、SSL、集群策略等）通常在 `/etc/rabbitmq/advanced.config` 中完成。

**环境变量文件**

另一个重要的配置文件是 `/etc/rabbitmq/rabbitmq-env.conf`，用于设置环境变量。

```bash
sudo vim /etc/rabbitmq/rabbitmq-env.conf
```

示例内容

```bash
# 设置节点的名称，默认为 rabbit@$(hostname)
# NODENAME=rabbit@rabbitmq-node01

# 告诉 Erlang 哪些 Cookie 用于节点间认证。集群中所有节点必须相同！
# ERLANG_COOKIE='YourSuperSecretCookieHere'
```

`ERLANG_COOKIE` 是集群的关键，在生产环境中设置一个长且复杂的随机字符串，并严格保密。

**日志查看**

RabbitMQ 日志默认位于 `/var/log/rabbitmq/`。遇到问题时首先查看这里。

```bash
tail -f /var/log/rabbitmq/rabbit@$(hostname).log
```

## 6、验证

**检查服务状态**

```bash
sudo systemctl status rabbitmq-server
sudo rabbitmqctl status # 更详细的状态信息
```

**访问管理控制台**

打开浏览器，访问 `http://<你的服务器IP地址>:15672`。

- 使用你创建的 **`prod_admin`** 用户和密码登录。
- 你应该能看到完整的仪表板，包含连接、通道、交换器、队列等各种信息。

**备份**

- **备份配置和数据**：定期备份 `/etc/rabbitmq/` 目录和 `/var/lib/rabbitmq/` 目录（存放数据）。
- **设置监控**：使用 Prometheus（配合 `rabbitmq_prometheus` 插件）或 Zabbix 等工具监控 RabbitMQ 的健康状态和性能指标。
- **配置集群**：如果需要高可用和扩展性，可以部署多节点 RabbitMQ 集群。
- **启用 TLS/SSL**：为客户端连接和管理界面启用加密，提升安全性。