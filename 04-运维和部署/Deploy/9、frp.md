# 二进制包安装

**1、下载**

https://github.com/fatedier/frp

releases 找到对应的 linux 版本和 windows 版本

# frp

## 服务器配置

1、修改 frps.toml

```bash
bindPort = 7000
vhostHttpPort = 8080
subdomainHost = "116.198.24.98"

webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "admin123"
```

> bindPort：客户端与服务端通信端口，必须
>
> vhostHttpPort：公网访问 HTTP 服务时使用的端口，必须
>
> subdomainHost：子域名模式，填写域名或者公网ip
>
> webServer.addr：控制台监听地址，非必须
>
> webServer.port：服务端 Dashboard 控制台端口，非必须

2、启动服务

```bash
./frps -c frps.toml
```

## 客户端配置

1、修改 frpc.toml

```bash
[common]
server_addr = 116.198.24.98
server_port = 7000

[tcp_3306]
type = tcp
local_ip = 127.0.0.1
local_port = 3306
remote_port = 6000

[web_8282]
type = http
local_port = 8282
custom_domains = 116.198.24.98
```

以上配置为

tcp 穿透，本地3306端口映射到服务器的6000端口

http 穿透，本地的8282端口直接映射到服务器的 vhostHttpPort 端口，此案例直接访问 116.198.24.98:8080

2、客户端启动

```bash
frpc.exe -c frpc.toml
```

## 有域名情况

1、服务端配置中，subdomainHost 改为域名

```bash
[common]
bind_port = 7000
vhostHttpPort = 8080
subdomain_host = example.com
```

2、服务端配置

```bash
[web_8383]
type = http
local_port = 8282
subdomain = web8383
```

3、此时访问地址将成为

http://web8383.example.com:8080