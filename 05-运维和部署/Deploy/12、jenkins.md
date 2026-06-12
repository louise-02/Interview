# Jenkins 简介

[Jenkins 官网](https://www.jenkins.io) | [文档](https://www.jenkins.io/doc/)

Jenkins 是开源 **CI/CD** 工具，用于自动化构建、测试、部署流水线。

| 功能 | 说明 |
| ---- | ---- |
| Pipeline | Jenkinsfile 定义流水线 |
| 插件 | 集成 Git、Maven、Docker、K8s 等 |
| 分布式构建 | Agent 节点分担任务 |

常用端口：

| 端口 | 说明 |
| ---- | ---- |
| 8080 | Web 界面 |

## 部署方式推荐

| 环境 | 推荐方式 | 说明 |
| ---- | -------- | ---- |
| **生产 / 日常使用** | **Docker** | 业界常见做法，数据挂载到 `/data/docker/jenkins/data` |
| 大规模 CI | Docker + Agent | 主节点 + 构建 Agent 扩展 |

---

# docker 安装（Jenkins LTS JDK17）

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)，挂载目录见 [docker/0、目录规划.md](./docker/0、目录规划.md)  
> 偏生产检查清单见 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md)

## 1、创建目录

```bash
mkdir -p /data/docker/jenkins/data
```

## 2、偏生产说明

Jenkins 以 Docker 为主，无单独 `conf/` 示例。偏生产重点：**强 admin 密码**、Nginx 前置 HTTPS、定期备份 `/data/docker/jenkins/data`，见下文 [Jenkins 配置与运维](#jenkins-配置与运维) 与 [docker/2、偏生产部署说明.md](./docker/2、偏生产部署说明.md)。

## 3、启动

```bash
cd /path/to/Deploy/docker
docker compose -f jenkins.yml up -d
```

## 4、获取初始密码

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

浏览器访问 `http://宿主机IP:8080`，输入初始密码完成向导。

## 5、常用操作

```bash
docker compose -f jenkins.yml ps
docker compose -f jenkins.yml logs -f
docker compose -f jenkins.yml down
```

> 注意：Jenkins 默认占用 8080 端口，与 Kafka UI 冲突，同时部署时需修改其中一个端口映射。

# Jenkins 配置与运维

## 1、初始化

- 安装推荐插件，创建 admin 账号（不用默认密码流程中的弱密码）
- **Manage Jenkins → Security** 启用「Prevent Cross Site Request Forgery exploits」
- 限制匿名用户权限，启用基于角色的授权

## 2、反向代理（生产推荐）

Nginx 前置 HTTPS，Jenkins 中设置 **Manage Jenkins → System → Jenkins URL** 为外网 HTTPS 地址：

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

## 3、备份

```bash
# 整个 jenkins_home 即全部配置、任务、凭据
tar -czf /backup/jenkins_$(date +%F).tar.gz /data/docker/jenkins/data
```

> 凭据加密与 master.key 在同一目录，备份需完整保留。

## 4、运行检查

```bash
docker compose -f jenkins.yml ps
curl -I http://127.0.0.1:8080/login
# 查看构建日志、磁盘占用
du -sh /data/docker/jenkins/data
```
