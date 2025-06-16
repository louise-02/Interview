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
mkdir -p /opt/src && cd /opt/src

# 下载
wget http://downloads.sourceforge.net/project/pcre/pcre/8.35/pcre-8.35.tar.gz

# 解压
tar zxvf pcre-8.35.tar.gz

# 编译安装
cd pcre-8.35
./configure --prefix=/opt/pcre/8.35
make && make install

# 查看版本
/opt/pcre/8.35/bin/pcre-config --version
```

## 2、安装 Nginx

下载地址：https://nginx.org/en/download.html

```bash
# 下载
mkdir -p /opt/src && cd /opt/src
wget http://nginx.org/download/nginx-1.26.2.tar.gz

# 解压
tar zxvf nginx-1.26.2.tar.gz

# 编译安装
cd nginx-1.26.2
./configure --prefix=/opt/nginx/1.26.2 --with-http_stub_status_module --with-http_ssl_module --with-pcre=/opt/src/pcre-8.35
make
make install

# 设置 current 软链接
ln -sfn /opt/nginx/1.26.2 /opt/nginx/current

# 创建对应数据目录结构
mkdir -p /data/nginx/1.26.2/{logs,run,tmp}
ln -sfn /data/nginx/1.26.2 /data/nginx/current

# 拷贝默认配置文件
mv /opt/nginx/1.26.2/conf /data/nginx/1.26.2/
mv /opt/nginx/1.26.2/html /data/nginx/1.26.2/

# 查看版本
/opt/nginx/current/sbin/nginx -v
```

## 3、配置 Nginx

修改配置文件

```bash
vi /data/nginx/current/conf/nginx.conf
```

配置文件示例

```bash
# 全局配置块
user  nginx;                      # 运行用户
worker_processes  auto;           # 工作进程数 (auto=自动匹配CPU核心数)
worker_rlimit_nofile 65535;       # 单个进程最大打开文件数(需要与系统 ulimit 设置匹配)
pid run/nginx.pid;    # 进程PID文件

events {
    # 使用 epoll 模型（Linux 下的高性能 IO 模型）
	use epoll;
	# 每个 worker 支持的最大连接数（总连接数 = worker_processes * worker_connections）
    worker_connections  16384;
    # 启用多个连接请求一并接收，提高并发能力
    multi_accept on;
}


http {
    # 加载 MIME 类型定义文件
    include       mime.types;
    # 默认的 MIME 类型（当无法识别时）
    default_type  application/octet-stream;
    
    # ===================== 临时目录配置 =====================
    client_body_temp_path tmp/client_body;
    proxy_temp_path       tmp/proxy;
    fastcgi_temp_path     tmp/fastcgi;
    scgi_temp_path        tmp/scgi;
    uwsgi_temp_path       tmp/uwsgi;
    
    # ===================== 日志配置 =====================
    # 访问日志路径与格式
    access_log  logs/access.log main;
    # 错误日志路径与日志级别（debug | info | notice | warn | error | crit）
    error_log   logs/error.log error;
    # 日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    
    # ===================== 连接性能优化 =====================
    # 启用 sendfile 提高文件传输效率
    sendfile        on;
    # 减少网络报文（延迟发送大数据块，优化文件传输）
    tcp_nopush     on;
    # 立即发送小数据包，适合动态请求
    tcp_nodelay    on;
    # 连接保持时长（秒）
    keepalive_timeout  65;
    # 每个连接最多处理的请求数（适用于长连接）
    keepalive_requests 10000;
    # 哈希表大小：用于加速查找 MIME 类型
    types_hash_max_size 2048;
    
    # ===================== Gzip 压缩优化 =====================
    gzip on;
    # 禁用 IE6 gzip（防兼容问题）
    gzip_disable "msie6";
    # 添加 Vary: Accept-Encoding 响应头
    gzip_vary on;
    # 支持代理请求压缩
    gzip_proxied any;
    # 压缩级别（1-9，越高越耗 CPU）
    gzip_comp_level 6;
    # 压缩使用的缓冲区
    gzip_buffers 16 8k;
    # 最小压缩长度（小于此值的不压缩）
    gzip_min_length 1k;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    
    # ===================== 缓冲配置 =====================
    # 客户端请求体缓冲区大小
    client_body_buffer_size 512k;
    # 上传文件大小限制
    client_max_body_size 50m;
    # 客户端请求头缓冲区
    client_header_buffer_size 64k;
    # 支持大请求头（如大 cookie）
    large_client_header_buffers 4 64k;
    
    # ===================== 超时设置 =====================
    # 响应超时
    send_timeout 60;
    # 请求体读取超时
    client_body_timeout 60;
    # 请求头读取超时
    client_header_timeout 60;
    
    
    # 隐藏Nginx版本号 (安全建议)
    server_tokens off;
    
    # ===================== 后端负载均衡配置 =====================
    upstream backend {
        # 三台后端应用服务，支持负载均衡
        server 192.168.1.101:8080 max_fails=3 fail_timeout=30s;
        server 192.168.1.102:8080 max_fails=3 fail_timeout=30s;
        server 192.168.1.103:8080 max_fails=3 fail_timeout=30s;

        # 启用 keepalive 长连接，减少连接频繁创建销毁
        keepalive 64;
    }

    # ===================== 主 server 配置 =====================
    server {
        # 监听 80 端口
        listen       80;
        # 虚拟主机名（支持多个，用空格分隔）
        server_name  localhost;

        # 前端项目地址
        location / {
            # 访问运行目录的 html 文件夹
            root   html;
            # 默认访问 index.html 或 index.htm
            index  index.html index.htm;
            if ($request_filename ~* .*.(html|htm)$) {
                expires    -1s;
                add_header Cache-Control "no-cache, no-store, private, must-revalidate, proxy-revalidate";
            }
            # 如果请求的文件不存在，使用 rewrite 把路径重写到 /index.html
            if (!-e $request_filename){
                rewrite ^/(.*) /index.html last;
            }
        }
        
        # 所有业务流量代理给 upstream 后端
        location /api {
            proxy_pass http://backend;
            
            # 透传头部信息给后端（用于识别来源）
            # 把客户端请求的 Host 头（例如访问 example.com）原样传给后端
            proxy_set_header Host $host;
            # 将客户端的真实 IP 写入 X-Real-IP 头
            proxy_set_header  X-Real-IP        $remote_addr;
            # 给 X-Forwarded-For 头加上客户端 IP
            proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
            # 传递一个标识头给后端，说明这是经过 Nginx 转发的请求,不是标准头，一般后端可据此做自定义逻辑
            proxy_set_header  X-NginX-Proxy true;
            
            # 连接超时设置
            # 与后端服务器建立连接的最大等待时间，超时直接失败
            proxy_connect_timeout 10s;
            # 向后端发送请求时的超时时间
            proxy_send_timeout 60s;
            # 等待后端响应内容的超时时间
            proxy_read_timeout 60s;
            
            # 启用缓冲，提升高并发性能
            # 开启缓冲：Nginx 把后端响应先存入缓冲区，再发送给客户端,避免后端慢速输出卡住连接
            proxy_buffering on;
            # 设置 16 个 64KB 的缓冲区，用于存放后端的响应内容
            proxy_buffers 16 64k;
            # 响应内容达到多少（128k）会立即开始写入客户端，不等待所有缓冲满
            proxy_busy_buffers_size 128k;
            # 当缓冲区满了，会将响应内容写入临时文件
            proxy_temp_file_write_size 128k;
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
/opt/nginx/current/sbin/nginx -t -p /data/nginx/current/ -c conf/nginx.conf
```

## 4、用户和目录

```bash
# 创建 nginx 用户和组（无登录权限）
sudo useradd -r -s /sbin/nologin nginx

# 授权数据目录
chown -R nginx:nginx /opt/nginx/
chown -R nginx:nginx /data/nginx/
```

## 5、启停 Nginx

启动

```bash
/opt/nginx/current/sbin/nginx -p /data/nginx/current/ -c conf/nginx.conf
```

> -p：指定 nginx 工作前缀目录

重新载入配置文件

```bash
/opt/nginx/current/sbin/nginx -s reload
```

重启

```bash
/opt/nginx/current/sbin/nginx -s reload -p /data/nginx/current/ -c conf/nginx.conf
```

停止

```bash
/opt/nginx/current/sbin/nginx -s quit -p /data/nginx/current/ -c conf/nginx.conf
```

## 6、注册服务

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
ExecStart=/opt/nginx/current/sbin/nginx -p /data/nginx/current/ -c /data/nginx/current/conf/nginx.conf
ExecReload=/opt/nginx/current/sbin/nginx -s reload -p /data/nginx/current/ -c /data/nginx/current/conf/nginx.conf
ExecStop=/opt/nginx/current/sbin/nginx -s quit -p /data/nginx/current/ -c /data/nginx/current/conf/nginx.conf
PIDFile=/data/nginx/current/run/nginx.pid
LimitNOFILE=65536
TimeoutStartSec=30
TimeoutStopSec=30
Restart=on-failure
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

## 7、多版本切换

```bash
ln -sfn /opt/nginx/1.27.0 /opt/nginx/current
ln -sfn /data/nginx/1.27.0 /data/nginx/current

# 重启服务即可应用新版本
sudo systemctl restart nginx
```

## 8、环境变量

```bash
echo 'export PATH=/opt/nginx/current/sbin:$PATH' >> /etc/profile.d/nginx.sh

chmod +x /etc/profile.d/nginx.sh

source /etc/profile.d/nginx.sh
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

假设没有指定 `--prefix`，默认路径为 `/usr/local/nginx` 

| 类型       | 路径                               | 说明                                       |
| ---------- | ---------------------------------- | ------------------------------------------ |
| 📄 主程序   | `/usr/local/nginx/sbin/nginx`      | Nginx 可执行文件                           |
| 📁 配置文件 | `/usr/local/nginx/conf/nginx.conf` | 主配置目录，包含 `nginx.conf` 和 `conf.d/` |
| 📁 默认网页 | `/usr/local/nginx/html/`           | 默认网页目录，含 `index.html` 等           |
| 📁 日志目录 | `/usr/local/nginx/logs/`           | 存放访问日志和错误日志                     |