# 1、yum 安装

## 1.1、安装

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

## 1.2、卸载

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

## 1.3、说明

| 类型       | 路径                                               | 说明                     |
| ---------- | -------------------------------------------------- | ------------------------ |
| 安装路径   | /usr/lib/jvm/java-1.8.0-openjdk-<version>/         | JDK 1.8 的安装目录       |
| 可执行文件 | /usr/lib/jvm/java-1.8.0-openjdk-<version>/bin/     | JDK 1.8 的可执行文件目录 |
| 环境变量   | /usr/lib/jvm/java-1.8.0-openjdk-<version>/         | JAVA_HOME                |
| 环境变量   | /usr/lib/jvm/java-1.8.0-openjdk-<version>/bin/     | PATH                     |
| 库文件     | /usr/lib/jvm/java-1.8.0-openjdk-<version>/jre/lib/ |                          |
| 符号链接   | /usr/bin/java                                      |                          |
| 符号链接   | /usr/bin/javac                                     |                          |

2、