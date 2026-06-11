# 1、Docker 介绍

官网地址：https://www.docker.com

Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的**容器**中，然后发布到任何流行的 Linux 机器上。

容器是完全使用沙箱机制，相互隔离。

![image-20250425093301603](pictures/image-20250425093301603.png)

## 1.1、镜像 Image

就是一个只读模板，比如：一个镜像可以包含一个完整的 CentOS，里面仅安装jdk或用户的其他应用。

## 1.2、容器 Container

镜像和容器的关系，就像是面向对象程序设计中的类和对象一样。

容器是从镜像创建的运行实例，它可以被启动、停止、 删除。

每个容器都是相互隔离的、保证安全的平台。

可以把容器看做是一个简易版的 Linux 环境（包括 root 用户权限、进程空间、用户空间）和运行在其中的应用程序。

## 1.3、仓库 Repository

仓库是集中存放镜像文件的场所。

# 2、Docker 安装

## 2.1、安装

```bash
# 1、yum 包更新到最新 
yum update

# 2、安装需要的软件包， yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的 
yum install -y yum-utils device-mapper-persistent-data lvm2

# 3、设置yum源
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo（中央仓库）
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo（阿里仓库）
# 4、安装docker，出现输入的界面都按 y 
yum install -y docker-ce

# 5、查看docker版本，验证是否验证成功
docker -v

# 6、启动docker环境
systemctl start docker

# 7、设置开机自启动
systemctl enable docker
```

## 2.2、配置镜像加速器

```bash
# 1、创建或修改文件
vi /etc/docker/daemon.json

# 2、添加以下内容
{
 "registry-mirrors":["https://docker.1ms.run"]
}

# 3、重启docker
systemctl restart docker

# 4、查看是否配置成功 Registry Mirrors
docker info
```

## 2.3、启停命令

```bash
# 启动服务
systemctl start docker 

# 停止服务 
systemctl stop docker

# 重启服务	
systemctl restart docker

# 查看服务的状态	
systemctl status docker 

# 设置开机自启动 
systemctl enable docker
```

## 2.4、卸载

```bash
# 停止 Docker 服务
sudo systemctl stop docker
sudo systemctl disable docker

# 卸载 Docker 软件包
sudo yum remove -y \
    docker-ce \           # Docker 社区版主程序
    docker-ce-cli \       # Docker 命令行工具
    containerd.io \       # 容器运行时
    docker-buildx-plugin  # Docker 多架构构建插件（如果存在）


# 删除残留数据和配置文件
镜像、容器、卷等数据默认存储在 /var/lib/docker
sudo rm -rf /var/lib/docker     # Docker 主数据目录
sudo rm -rf /var/lib/containerd # containerd 数据目录
sudo rm -rf /etc/docker         # Docker 配置文件目录

# 移除 Docker 官方仓库
sudo rm -f /etc/yum.repos.d/docker-ce.repo
```



# 3、Docker 命令

## 3.1、镜像相关

```bash
# 查看本地镜像 
docker images

# 搜索镜像仓库，推荐：https://hub.docker.com/
docker search 镜像名称
# 搜索镜像仓库 指定镜像地址 镜像地址/镜像名称
docker search docker.1ms.run/mysql

# 拉取镜像 没有tag默认latest
docker pull 镜像名称[:tag]
# 拉取镜像，指定镜像地址 镜像地址/镜像名称[:tag]
docker pull docker.1ms.run/mysql:8.0.32

# 删除镜像
docker rmi 镜像名称[:tag]

# 镜像保存为本地文件
docker save -o 文件名.tar 镜像名称
docker save -o fastdfs.tar delron/fastdfs

# 加载本地文件
docker load -i 文件名.tar
docker load -i fastdfs.tar
```

## 3.2、容器相关

```bash
# 查看本地容器
docker ps 	  # 能查看正在运行
docker ps -a  # 能查看所有的容器（运行的和停止的）
docker ps -qa   # 只查询id

# 创建一个新的容器并运行（交互式）
docker run -it --name=容器名 镜像名称 /bin/bash
# 创建一个新的容器并运行（守护式）
docker run -d --name=容器名 镜像名称
# 进入容器内部
docker exec -it 容器名称/容器id /bin/bash

# 启动容器
docker start 容器名称/容器id

# 停止容器
docker stop 容器名称/容器id
# 批量停止
docker stop `docker ps -qa`

# 删除容器
docker rm 容器名称/容器id

# 查看容器信息
docker inspect 容器名称/容器id

# 修改为开机自启
docker update --restart=always 容器名称/容器id
# 修改为非开机自启
docker update --restart=no 容器名称/容器id
# 批量设置开机自启
docker update --restart=always $(docker ps -aq)
```

# 4、Docker 应用部署

## 4.1、Redis

```bash
# 搜索镜像
docker search redis
docker search docker.1ms.run/redis

# 拉取镜像
docker pull redis:7.4.2
# 拉取镜像 指定镜像源
docker pull docker.1ms.run/redis:7.4.2

# 指定镜像源修改镜像名
docker tag docker.1ms.run/redis:7.4.2 redis:7.4.2
docker rmi docker.1ms.run/redis:7.4.2

# 简易版运行容器
docker run -d \
  --name redis7.4.2 \
  -p 6379:6379 \
  -v /opt/redis/data:/data \
  redis:7.4.2 \
  redis-server --appendonly yes

# 运行容器 参数详情
docker run -d \
  --name redis7.4.2 \
  -p 6379:6379 \
  -v /opt/redis/data:/data \          # 持久化数据到宿主机
  -v /opt/redis/conf/redis.conf:/usr/local/etc/redis/redis.conf \  # 挂载自定义配置
  --memory 512m \                    # 限制内存为 512MB
  --memory-swap 1g \                 # 内存+Swap 总大小 1GB
  --restart unless-stopped \         # 自动重启
  redis:7.4.2 \                     # 使用最新版镜像
  redis-server /usr/local/etc/redis/redis.conf # 指定配置文件
```

配置文件：/opt/redis/conf/redis.conf

```bash
# 设置访问密码
requirepass your_secure_password_here

# 其他配置（可选）
bind 0.0.0.0
protected-mode no
appendonly yes
```

## 4.2、Mysql

```bash
# 搜索镜像
docker search mysql
docker search docker.1ms.run/mysql

# 拉取镜像
docker pull mysql:8.0.32
# 拉取镜像 指定镜像源
docker pull docker.1ms.run/mysql:8.0.32

# 指定镜像源修改镜像名
docker tag docker.1ms.run/mysql:8.0.32 mysql:8.0.32
docker rmi docker.1ms.run/mysql:8.0.32

# 简易版运行容器
docker run -d \
  --name mysql8.0.32 \
  -p 3306:3306 \
  -v /opt/mysql8.0.32/data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=123456 \
  mysql:8.0.32 \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci
  
# 设置容器自启动
docker update --restart=always mysql8.0.32
  
# 运行容器 参数详情
docker run -d \
  --name mysql8.0.32 \
  -p 3306:3306 \                                          # 映射端口
  -v /opt/mysql8.0.32/data:/var/lib/mysql \               # 持久化数据
  -v /opt/mysql8.0.32/conf.d:/etc/mysql/conf.d \          # 自定义配置
  -v /opt/mysql8.0.32/init:/docker-entrypoint-initdb.d \  # 初始化脚本
  -e MYSQL_ROOT_PASSWORD=123456 \                         # root密码
  -e MYSQL_DATABASE=mydb \                                # 初始数据库
  -e MYSQL_USER=app_user \                                # 普通用户
  -e MYSQL_PASSWORD=app_password \                        # 普通用户密码
  -e TZ=Asia/Shanghai \                                   # 时区
  mysql:8.0.32 \                                          # 指定镜像版本
  --character-set-server=utf8mb4 \                        # 直接传递参数给mysqld
  --collation-server=utf8mb4_unicode_ci
```

## 4.3、Nacos

```bash
# 下载 nacos-docker 项目
git clone https://github.com/nacos-group/nacos-docker.git
cd nacos-docker

# 执行 docker-compose 命令启动Nacos 此启动方式会启动 grafana 和 prometheus
docker-compose -f example/standalone-derby.yaml up

# 编写简易版 example 中创建 docker-compose.yaml
version: "3"
services:
  nacos:
    image: nacos/nacos-server:${NACOS_VERSION} # version 也可手动指定 指的镜像版本
    container_name: nacos-standalone
    ports:
      - "8848:8848"    # Nacos UI 端口
      - "9848:9848"    # gRPC 通信端口（可选）
      - "9849:9849"    # gRPC 通信端口（可选）
    environment:
      - MODE=standalone
      - PREFER_HOST_MODE=hostname
      - SPRING_DATASOURCE_PLATFORM=derby
    restart: unless-stopped

# 启动简易版
docker-compose -f docker-compose.yaml up

# 编写mysql版  example 中创建 docker-mysql-compose.yaml
version: "3"
services:
  nacos:
    image: nacos/nacos-server:${NACOS_VERSION} # version 也可手动指定 指的镜像版本
    container_name: nacos-mysql
    ports:
      - "8848:8848"
    environment:
      - MODE=standalone
      - PREFER_HOST_MODE=hostname
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=your-mysql-host
      - MYSQL_SERVICE_DB_NAME=nacos_config
      - MYSQL_SERVICE_PORT=3306
      - MYSQL_SERVICE_USER=nacos
      - MYSQL_SERVICE_PASSWORD=nacos123
    restart: unless-stopped


# 验证Nacos服务是否启动成功
docker logs -f $container_id
Nacos started successfully in xxxx mode. use xxxx storage
```

> SPRING_DATASOURCE_PLATFORM：持久化方式
>
> - derby：使用内嵌数据库 Derby，适合开发或测试环境
> - mysql：使用 MySQL，适合生产环境
>
> PREFER_HOST_MODE：用于注册服务时确定节点自身的地址
>
> - hostname：使用主机名注册实例（推荐 Docker/K8s 环境）
> - ip：使用容器的 IP 地址进行注册（不推荐容器中使用）

​	

# 5、Docker Image Build 镜像构建



# 6、Docker Compose 编排

```yml
# =========================
# 版本声明（新版已不强制）
# =========================
version: "3.9"   # 可选：3.7 / 3.8 / 3.9（推荐 3.8+）

# =========================
# 服务定义（核心）
# =========================
services:

  # -------------------------
  # 示例：Web 应用服务
  # -------------------------
  app:
    container_name: my-app   # ❗指定容器名（不推荐多实例时使用）
    
    # ===== 镜像相关（二选一）=====
    image: nginx:latest      # 方式1：直接使用镜像
    # build:                  # 方式2：本地构建
    #   context: .           # Dockerfile 所在目录
    #   dockerfile: Dockerfile
    #   args:                # 构建参数
    #     VERSION: "1.0"

    # ===== 端口映射 =====
    ports:
      - "8080:80"           # 宿主机:容器
      # - "80:80"           # 可选多个端口

    # ===== 环境变量 =====
    environment:
      - TZ=Asia/Shanghai
      - APP_ENV=dev
      # 也可以 map 形式
      # DB_HOST: mysql

    # 或者使用 env 文件（推荐）
    # env_file:
    #   - .env

    # ===== 启动命令 =====
    command: ["nginx", "-g", "daemon off;"]
    # entrypoint: ["/bin/sh", "-c"]  # 会覆盖镜像默认入口（慎用）

    # ===== 数据挂载 =====
    volumes:
      - ./html:/usr/share/nginx/html   # 本地目录挂载
      - app-data:/data                 # 命名卷
      # - /host/path:/container/path:ro  # 只读挂载（ro）

    # ===== 网络 =====
    networks:
      - app-network

    # ===== 依赖服务（启动顺序）=====
    depends_on:
      - mysql
      # 注意：只控制启动顺序，不保证“已就绪”

    # ===== 健康检查 =====
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s     # 检查间隔
      timeout: 5s       # 超时
      retries: 3        # 重试次数
      start_period: 10s # 启动宽限期

    # ===== 重启策略 =====
    restart: always
    # 可选：
    # no               不重启
    # always           总是重启
    # on-failure       失败才重启
    # unless-stopped   手动停止才不重启（常用）

    # ===== 日志 =====
    logging:
      driver: "json-file"   # json-file / syslog / fluentd
      options:
        max-size: "10m"
        max-file: "3"

    # ===== 资源限制（Swarm 才完全生效）=====
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M

    # ===== 访问控制 / 用户 =====
    user: "1000:1000"  # 指定运行用户

    # ===== tty / stdin =====
    tty: true
    stdin_open: true


  # -------------------------
  # MySQL 服务示例
  # -------------------------
  mysql:
    image: mysql:8.0
    container_name: mysql8

    ports:
      - "3306:3306"

    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: test

    volumes:
      - mysql-data:/var/lib/mysql

    command:
      # 多种启动参数示例
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_general_ci

    restart: unless-stopped

    networks:
      - app-network

    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 10s
      retries: 5


  # -------------------------
  # Redis 示例
  # -------------------------
  redis:
    image: redis:7
    ports:
      - "6379:6379"

    command: ["redis-server", "--appendonly", "yes"]

    volumes:
      - redis-data:/data

    restart: always

    networks:
      - app-network


# =========================
# 网络定义
# =========================
networks:
  app-network:
    driver: bridge   # 可选：
                     # bridge（默认）
                     # host（共享宿主机网络）
                     # none（无网络）
    ipam:
      config:
        - subnet: 172.20.0.0/16  # 自定义网段


# =========================
# 数据卷定义
# =========================
volumes:
  mysql-data:
    driver: local

  redis-data:
    driver: local

  app-data:
    driver: local


# =========================
# 配置（configs）——Swarm 用
# =========================
configs:
  app_config:
    file: ./config/app.yml


# =========================
# 密钥（secrets）——Swarm 用
# =========================
secrets:
  db_password:
    file: ./secrets/db_password.txt
```

