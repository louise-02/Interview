# yum 安装

## 1、安装

安装 EPEL 仓库

```bash
sudo yum install epel-release
```

安装 Nginx

```bash
sudo yum install nginx
```

启动 Nginx 服务

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

检查 Nginx 服务是否正在运行

```bash
sudo systemctl status nginx
```

调整防火墙设置

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

## 2、卸载

停止 Nginx 服务

```bash
sudo systemctl stop nginx
```

卸载 Nginx

```bash
sudo yum remove nginx
```

清理残留的配置文件和数据

```bash
sudo rm -rf /etc/nginx
sudo rm -rf /var/log/nginx
sudo rm -rf /usr/share/nginx
```

检查 Nginx 是否已卸载

```bash
nginx -v
```

# 编译 安装

## 1、安装库

**安装编译工具及库文件**

```bash
yum -y install make zlib zlib-devel gcc-c++ libtool  openssl openssl-devel
```

**安装 PCRE**

PCRE 作用是让 Nginx 支持 Rewrite 功能。

```bash
cd /usr/local/src/

//下载
wget http://downloads.sourceforge.net/project/pcre/pcre/8.35/pcre-8.35.tar.gz

//解压
tar zxvf pcre-8.35.tar.gz

//编译安装
cd pcre-8.35
./configure
make && make install

//查看版本
pcre-config --version
```

## 2、安装 Nginx

下载地址：https://nginx.org/en/download.html

```bash
//下载
cd /usr/local/src/
wget http://nginx.org/download/nginx-1.26.2.tar.gz

//解压
tar zxvf nginx-1.26.2.tar.gz

//编译安装
cd nginx-1.26.2
./configure --prefix=/usr/local/webserver/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre=/usr/local/src/pcre-8.35
make
make install

//查看版本
/usr/local/webserver/nginx/sbin/nginx -v
```

## 3、配置 Nginx

修改配置文件

```bash
vi /usr/local/webserver/nginx/conf/nginx.conf
```

配置文件示例

```bash
# 全局配置块
user  nginx;                      # 运行用户
worker_processes  auto;           # 工作进程数 (auto=自动匹配CPU核心数)
error_log  /var/log/nginx/error.log warn;  # 错误日志路径
pid        /var/run/nginx.pid;    # 进程PID文件

events {
    worker_connections  1024;     # 单个工作进程最大连接数
    multi_accept on;              # 允许同时接受多个连接
}


http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;           # 启用高效文件传输模式
    tcp_nopush     on;            # 优化数据包发送
    tcp_nodelay    on;            # 禁用Nagle算法
    keepalive_timeout  65;        # 长连接超时时间(秒)
    server_tokens off;            # 隐藏Nginx版本号 (安全建议)
    client_max_body_size 100M;    # 允许上传的最大文件大小

    # 日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;  # 访问日志路径

    # Gzip压缩配置
    gzip on;
    gzip_min_length 1k;           # 最小压缩文件大小
    gzip_comp_level 6;            # 压缩级别(1-9)
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_vary on;                 # 根据客户端支持情况启用压缩
    
    # 定义上游服务器 (负载均衡示例)
    upstream backend {
        server 127.0.0.1:8282/guns/ weight=5;  # 本地应用服务1，权重5
        server 10.108.5.203:8282/guns/;       # 其他服务器
        keepalive 32;            # 保持的长连接数
    }

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
            if ($request_filename ~* .*.(html|htm)$) {
                expires    -1s;
                add_header Cache-Control "no-cache, no-store, private, must-revalidate, proxy-revalidate";
            }
            if (!-e $request_filename){
                rewrite ^/(.*) /index.html last;
            }
        }
        
        location /api {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header  X-Real-IP        $remote_addr;
            proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_set_header X-NginX-Proxy true;
        }


        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
		
		# 禁止访问隐藏文件
        location ~ /\.(?!well-known) {
            deny all;
            access_log off;
            log_not_found off;
        }
    }

}
```

校验配置文件

```bash
/usr/local/webserver/nginx/sbin/nginx -t
```

## 4、启停 Nginx

启动

```bash
/usr/local/webserver/nginx/sbin/nginx
```

重新载入配置文件

```bash
/usr/local/webserver/nginx/sbin/nginx -s reload
```

重启

```bash
/usr/local/webserver/nginx/sbin/nginx -s reopen
```

停止

```bash
/usr/local/webserver/nginx/sbin/nginx -s stop
```

## 5、注册服务

修改配置

```bash
sudo vim /etc/systemd/system/nginx.service
```

内容如下

```bash
[Unit]
Description=Custom Nginx
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/webserver/nginx/sbin/nginx
ExecReload=/usr/local/webserver/nginx/sbin/nginx -s reload
ExecStop=/usr/local/webserver/nginx/sbin/nginx -s quit
PIDFile=/usr/local/webserver/nginx/logs/nginx.pid
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

启动服务

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reexec
sudo systemctl daemon-reload

# 设置开机自启
sudo systemctl enable nginx

# 启动 nginx
sudo systemctl start nginx

# 查看状态
systemctl status nginx

# 重新加载配置
sudo systemctl reload nginx
```

# nginx

## 1、yum 安装目录

| 类型           | 路径                                    | 说明                                       |
| -------------- | --------------------------------------- | ------------------------------------------ |
| 📄 主程序       | `/usr/sbin/nginx`                       | Nginx 可执行文件                           |
| 📁 配置文件     | `/etc/nginx/`                           | 主配置目录，包含 `nginx.conf` 和 `conf.d/` |
| 📁 默认网页     | `/usr/share/nginx/html/`                | 默认网页目录，含 `index.html` 等           |
| 📁 服务控制脚本 | `/usr/lib/systemd/system/nginx.service` | 用于 systemd 管理 Nginx                    |
| 📁 日志目录     | `/var/log/nginx/`                       | 存放访问日志和错误日志                     |
| 📁 模块库       | `/usr/lib64/nginx/`                     | 一些动态模块（`.so` 文件）所在目录         |
| 📁 缓存临时目录 | `/var/cache/nginx/`                     | Nginx 的临时缓存目录                       |

## 2、编译安装目录

假设没有指定 `--prefix`，默认路径为 `/usr/local/nginx` ，文件都在此目录，不污染系统其他位置。

| 类型       | 路径                               | 说明                                       |
| ---------- | ---------------------------------- | ------------------------------------------ |
| 📄 主程序   | `/usr/local/nginx/sbin/nginx`      | Nginx 可执行文件                           |
| 📁 配置文件 | `/usr/local/nginx/conf/nginx.conf` | 主配置目录，包含 `nginx.conf` 和 `conf.d/` |
| 📁 默认网页 | `/usr/local/nginx/html/`           | 默认网页目录，含 `index.html` 等           |
| 📁 日志目录 | `/usr/local/nginx/logs/`           | 存放访问日志和错误日志                     |