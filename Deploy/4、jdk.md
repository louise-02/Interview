# yum 安装

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

# 二进制包安装

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

