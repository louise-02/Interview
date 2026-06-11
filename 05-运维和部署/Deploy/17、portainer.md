# Portainer 简介

[Portainer 官网](https://www.portainer.io) | [GitHub](https://github.com/portainer/portainer)

Portainer CE（Community Edition）是 Docker 的可视化管理工具，通过 Web 界面管理容器、镜像、网络、数据卷和 Compose 应用。

| 功能 | 说明 |
| ---- | ---- |
| 容器管理 | 启停、重启、删除、查看日志、进入终端 |
| 镜像管理 | 查看本地镜像、删除无用镜像 |
| Stack | 通过 Web 部署 / 更新 docker-compose 应用 |
| 多主机 | 配合 Agent 管理多台 Docker 主机 |

Portainer **仅支持 Docker 部署**，无独立二进制安装包。

常用端口：

| 端口 | 说明 |
| ---- | ---- |
| 9000 | HTTP Web 界面 |
| 9443 | HTTPS Web 界面 |
| 9001 | Agent 通信端口（多主机场景） |

# docker 安装

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)

## 1、创建目录

```bash
mkdir -p /data/docker/portainer/data
```

## 2、拉取镜像

```bash
docker pull portainer/portainer-ce:2.21.4
```

## 3、启动

```bash
cd /path/to/Deploy/docker
docker compose -f portainer.yml up -d
```

或使用 docker run：

```bash
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /data/docker/portainer/data:/data \
  -e TZ=Asia/Shanghai \
  --restart unless-stopped \
  portainer/portainer-ce:2.21.4
```

## 4、离线传输镜像

```bash
# 有网机器
docker pull portainer/portainer-ce:2.21.4
docker save -o portainer-ce-2.21.4.tar portainer/portainer-ce:2.21.4

# 内网机器
docker load -i portainer-ce-2.21.4.tar
docker compose -f portainer.yml up -d
```

## 5、初始化

浏览器访问 `http://宿主机IP:9000`（或 `https://宿主机IP:9443`）。

1. 首次进入设置 **admin 管理员密码**（至少 12 位）
2. 选择 **Get Started** → 连接本机 Docker（Local）
3. 进入 Dashboard 即可管理当前主机上的容器

## 6、常用命令

```bash
docker compose -f docker/portainer.yml ps
docker compose -f docker/portainer.yml logs -f
docker compose -f docker/portainer.yml restart
docker compose -f docker/portainer.yml down
```

> 注意：`down` 会删除 Portainer 容器，但 `/data/docker/portainer/data` 中的配置和数据会保留，重新 `up -d` 即可恢复。

# 多主机管理（Agent）

Portainer Server 管理本机；其他 Docker 主机部署 **Agent**，由 Server 统一纳管。

## 1、在远程主机部署 Agent

```bash
mkdir -p /data/docker/portainer-agent

cd /path/to/Deploy/docker
docker compose -f portainer-agent.yml up -d
```

## 2、在 Portainer 中添加环境

Portainer 控制台 → **Environments** → **Add environment** → **Agent**

- Name：自定义名称（如 `node-02`）
- Environment address：`远程主机IP:9001`

添加成功后，可在 Portainer 中切换环境，管理不同主机上的容器。

## 3、Agent 离线镜像

```bash
docker pull portainer/agent:2.21.4
docker save -o portainer-agent-2.21.4.tar portainer/agent:2.21.4
```

# 常用操作

## 1、容器

**Containers** 列表 → 点击容器名称：

| 操作 | 说明 |
| ---- | ---- |
| Logs | 查看容器标准输出日志（等同 `docker logs -f`） |
| Console | 进入容器终端（等同 `docker exec -it`） |
| Stats | 查看 CPU、内存实时占用 |
| Start / Stop / Restart | 启停容器 |

快捷操作：列表页勾选容器 → 批量 Start / Stop / Remove。

## 2、镜像

**Images** 页面：

- 查看本地所有镜像及占用空间
- 点击 **Remove** 删除无用镜像
- 点击 **Import** 从 tar 包导入镜像（内网常用）

```bash
# 命令行导入后，Images 页面刷新即可看到
docker load -i my-app.tar
```

## 3、Stack（Compose 应用）

**Stacks** → **Add stack**：

1. 输入 Stack 名称
2. 粘贴 docker-compose.yml 内容
3. 点击 **Deploy the stack**

后续可在 Stack 详情中 **Editor** 修改 compose 内容并 **Update**，或 **Stop / Remove** 整个应用。

## 4、数据卷

**Volumes** 页面查看、创建、删除 Docker 卷。

绑定挂载（`-v /host/path:/container/path`）的目录不在此列表，需在宿主机上直接管理。

## 5、网络

**Networks** 页面查看 Docker 网络，创建自定义 bridge 网络，查看容器 IP。

## 6、清理资源

**Images** → 勾选 Unused → Remove

或使用 Portainer 首页 **Home** 中的资源概览，配合命令行：

```bash
docker system df
docker system prune -a    # 慎用，删除所有未使用的镜像
```

# Portainer 配置

## 1、目录说明

| 类型 | 宿主机路径 | 容器路径 | 说明 |
| ---- | ---------- | -------- | ---- |
| 数据目录 | `/data/docker/portainer/data` | `/data` | 用户、环境、Stack 等配置 |
| Docker 套接字 | `/var/run/docker.sock` | `/var/run/docker.sock` | 与 Docker 守护进程通信 |

## 2、修改 Web 端口

编辑 [docker/portainer.yml](./docker/portainer.yml)：

```yaml
ports:
  - "9100:9000"   # 宿主机 9100 映射到容器 9000
```

## 3、重置 admin 密码

忘记密码时，停止 Portainer 后删除 data 目录中的用户数据重新初始化（**会丢失已配置的 Stack 和环境**）：

```bash
docker compose -f portainer.yml down
sudo rm -rf /data/docker/portainer/data/*
docker compose -f portainer.yml up -d
# 重新访问 Web 设置 admin 密码
```

或保留数据仅重置密码（Portainer 2.x）：

```bash
docker stop portainer
docker run --rm -v /data/docker/portainer/data:/data portainer/helper-reset-password
docker start portainer
```

## 4、开放防火墙

```bash
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --reload
```

## 5、注意事项

| 项目 | 说明 |
| ---- | ---- |
| 权限 | 挂载 `docker.sock` 等同于 root 权限，勿暴露到公网 |
| 版本 | 离线环境请固定镜像 tag，如 `2.21.4`，避免用 `latest` |
| 资源 | Portainer 本身占用很小（约 50～100MB 内存） |
| 日志 | Logs 页只看容器 stdout/stderr，应用写入文件的日志需在宿主机或 Console 中查看 |
| 备份 | 定期备份 `/data/docker/portainer/data` |

## 6、常见问题

**页面无法访问**

```bash
docker ps | grep portainer
docker logs portainer --tail 50
ss -tlnp | grep 9000
```

**添加 Local 环境失败**

确认 Docker 服务运行中，且 Portainer 容器能访问 `/var/run/docker.sock`。

**Stack 部署失败**

在 Stack 详情 → **Logs** 查看错误；常见原因是镜像未导入、端口冲突、挂载目录不存在。

**Agent 连接不上**

检查远程主机 9001 端口是否开放，Agent 容器是否运行：

```bash
docker ps | grep portainer-agent
curl http://远程IP:9001/ping
```
