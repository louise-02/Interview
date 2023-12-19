# Linux基础

## 目录结构

## 基础命令

### cd 目录操作

`cd a` 进入当前目录下的a目录

`cd a/b` 进入当前目录下的a目录下的b目录

`cd /a/b` 进入跟目录下的a目录下的b目录

`cd ..` 返回上一级目录

`cd ~` root回到/root目录 其他用户tom回到/home/tom

`cd -` 返回上次工作的目录

`cd /` 返回根目录

### mkdir 创建目录

```

```



# 软件安装

## Nginx

**一、安装编译工具及库文件**

```
yum -y install make zlib zlib-devel gcc-c++ libtool  openssl openssl-devel
```

**二、安装 PCRE**

PCRE 作用是让 Nginx 支持 Rewrite 功能。

1、下载 PCRE 安装包，http://downloads.sourceforge.net/project/pcre/pcre/8.35/pcre-8.35.tar.gz

```
[root@bogon src]# cd /usr/local/src/
[root@bogon src]# wget http://downloads.sourceforge.net/project/pcre/pcre/8.35/pcre-8.35.tar.gz
```

2、解压安装包

```
[root@bogon src]# tar zxvf pcre-8.35.tar.gz
```

3、进入安装包目录

```
[root@bogon src]# cd pcre-8.35
```

4、编译安装 

```
[root@bogon pcre-8.35]# ./configure
[root@bogon pcre-8.35]# make && make install
```

5、查看pcre版本

```
[root@bogon pcre-8.35]# pcre-config --version
```

**三、安装Nginx**

1、下载 Nginx并解压，下载地址：https://nginx.org/en/download.html

```
[root@bogon src]# cd /usr/local/src/
[root@bogon src]# wget http://nginx.org/download/nginx-1.6.2.tar.gz
[root@bogon src]# tar zxvf nginx-1.6.2.tar.gz
```

2、进入安装包目录

```
[root@bogon src]# cd nginx-1.6.2
```

3、编译安装

- `--prefix=/usr/local/webserver/nginx`：指定 Nginx 的安装目录。
- `--with-http_stub_status_module`：启用 http_stub_status_module 模块，该模块提供了一个简单的状态页面，用于监控 Nginx 的运行状态和统计信息。
- `--with-http_ssl_module`：启用 http_ssl_module 模块，该模块提供了对 SSL/TLS 加密的支持，使 Nginx 能够处理 HTTPS 请求。
- `--with-pcre=/usr/local/src/pcre-8.35`：指定 PCRE（Perl Compatible Regular Expressions）库的路径。PCRE 是一个正则表达式库，用于处理正则表达式匹配。

```
[root@bogon nginx-1.6.2]# ./configure --prefix=/usr/local/webserver/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre=/usr/local/src/pcre-8.35
[root@bogon nginx-1.6.2]# make
[root@bogon nginx-1.6.2]# make install
```

4、查看nginx版本

```
[root@bogon nginx-1.6.2]# /usr/local/webserver/nginx/sbin/nginx -v
```

5、启动nginx

```
[root@bogon conf]# /usr/local/webserver/nginx/sbin/nginx
```

6、访问页面

`http://ip`

7、其他命令

```
/usr/local/webserver/nginx/sbin/nginx -s reload            # 重新载入配置文件
/usr/local/webserver/nginx/sbin/nginx -s reopen            # 重启 Nginx
/usr/local/webserver/nginx/sbin/nginx -s stop              # 停止 Nginx
```

