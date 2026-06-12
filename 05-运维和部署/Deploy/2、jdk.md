# JDK 简介

[Adoptium（OpenJDK）](https://adoptium.net) | [Oracle JDK](https://www.oracle.com/cn/java/technologies/downloads/)

JDK 是 Java 程序的运行环境。部署其他 Java 中间件（Kafka、Nacos、Elasticsearch 等）前通常需先安装 JDK。

| 版本 | 说明 |
| ---- | ---- |
| JDK 8 | 老项目、部分中间件仍使用 |
| JDK 11 | Kafka 3.x 推荐 |
| JDK 17+ | Nacos 2.x、Spring Boot 3.x 等 |

## 部署方式推荐

| 环境 | 推荐方式 | 说明 |
| ---- | -------- | ---- |
| **生产** | **yum 或二进制** | 按应用要求锁定版本；多服务共存时注意版本隔离 |
| 离线环境 | 二进制 | 上传 tar 包安装，路径可控 |

> 安装路径、环境变量规范见 [0、readme.md](./0、readme.md)。

---

# yum 安装（OpenJDK 8）

## 1、安装

安装 jdk

```bash
#CentOS
sudo yum install java-1.8.0-openjdk
#Ubuntu
sudo apt install openjdk-8-jdk -y
```

验证安装

```bash
java -version
```

## 2、卸载

```bash
sudo yum remove java-1.8.0-openjdk java-1.8.0-openjdk-headless
sudo yum remove java-1.8.0-openjdk*
清理残留
yum autoremove

查看已安装的版本
dpkg -l | grep openjdk
sudo apt purge openjdk-8-jdk openjdk-8-jre*
apt autoremove


openjdk-*
oracle-java*
```

## 3、说明

| 类型       | 路径                                               | 说明                     |
| ---------- | -------------------------------------------------- | ------------------------ |
| 安装路径   | /usr/lib/jvm/java-1.8.0-openjdk-<version>/         | JDK 1.8 的安装目录       |
| 可执行文件 | /usr/lib/jvm/java-1.8.0-openjdk-<version>/bin/     | JDK 1.8 的可执行文件目录 |
| 环境变量   | /usr/lib/jvm/java-1.8.0-openjdk-<version>/         | JAVA_HOME                |
| 环境变量   | /usr/lib/jvm/java-1.8.0-openjdk-<version>/bin/     | PATH                     |
| 库文件     | /usr/lib/jvm/java-1.8.0-openjdk-<version>/jre/lib/ |                          |
| 符号链接   | /usr/bin/java                                      |                          |
| 符号链接   | /usr/bin/javac                                     |                          |

# 二进制包安装（JDK 8u391）

## 0、jdk 下载

open jdk 下载地址

https://adoptium.net/zh-CN/download/

oracle jdk 下载地址

https://www.oracle.com/cn/java/technologies/downloads/

## 1、安装

上传 tar 包，以 `jdk-8u391-linux-x64.tar.gz` 为例

```bash
cd /opt/jdk

tar -zxvf jdk-8u391-linux-x64.tar.gz
```

## 2、配置环境变量

创建自己的配置文件

```bash
vi /etc/profile.d/my_env.sh
```

配置以下内容

```bash
#!/bin/bash

export JAVA_HOME=/opt/jdk/jdk1.8.0_391
export PATH=$JAVA_HOME/bin:$PATH
```

刷新配置

```bash
source /etc/profile
```

验证版本

```bash
java -version
```

# JDK 配置与运维

## 1、版本与环境

```bash
# 确认环境变量生效
echo $JAVA_HOME
which java

# 多版本切换
sudo alternatives --config java

# 生产建议：锁定版本，避免 yum update 意外升级
sudo yum install yum-plugin-versionlock
sudo yum versionlock add java-1.8.0-openjdk*
```
