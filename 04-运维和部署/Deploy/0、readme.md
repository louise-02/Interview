# 目录规划

## yum

| 文件类型   | 路径示例                             | 说明                                |
| ---------- | ------------------------------------ | ----------------------------------- |
| 可执行文件 | `/usr/bin/`、`/usr/sbin/`            | 用户/系统执行命令所在路径           |
| 配置文件   | `/etc/软件名/`                       | 主配置文件目录                      |
| 日志文件   | `/var/log/软件名/`                   | 各类日志输出目录                    |
| 数据文件   | `/var/lib/软件名/`                   | 运行过程中产生的长期数据            |
| 缓存文件   | `/var/cache/软件名/`                 | 可以重新生成的缓存数据              |
| 运行时文件 | `/run/软件名/` 或 `/var/run/软件名/` | PID 文件、Socket 文件、临时状态文件 |
| 服务脚本   | `/etc/systemd/system/`               | Systemd 服务文件                    |
| 库文件     | `/usr/lib/` 或 `/usr/lib64/`         | 动态库文件（.so）                   |
| 临时文件   | `/tmp/`、`var/tmp/`                  | 临时使用的数据文件（可随时清理）    |

## 编译或解压

使用软链接来控制多版本，以 redis 为例

```
/opt/redis/
├── 7.2.4/
│   └── bin/
└── current -> 7.2.4

/data/redis/
├── 7.2.4/
│   ├── conf/        ← redis.conf 等
│   ├── logs/        ← redis.log
│   ├── data/        ← dump.rdb / appendonly.aof
│   ├── tmp/         ← 临时文件（如 RDB 重写）
│   └── redis.pid
└── current -> 7.2.4
```

常用目录

| 文件类型   | 推荐路径示例                           | 说明                                |
| ---------- | -------------------------------------- | ----------------------------------- |
| 源码包     | `/opt/src/`                            | 用于存放各个软件源码包              |
| 可执行文件 | `/opt/<软件名>/<版本>/bin/`            | 用户/系统执行命令所在路径           |
| 配置文件   | `/data/<软件名>/<版本>/conf/`          | 主配置文件目录                      |
| 日志文件   | `/data/<软件名>/<版本>/logs/`          | 各类日志输出目录                    |
| 数据文件   | `/data/<软件名>/<版本>/data/`          | 运行过程中产生的长期数据            |
| 缓存文件   | `/data/<软件名>/<版本>/cache/`（如有） | 可以重新生成的缓存数据              |
| 运行时文件 | `/data/<软件名>/<版本>/run/`           | PID 文件、Socket 文件、临时状态文件 |
| 服务脚本   | `/etc/systemd/system/<软件名>.service` | Systemd 服务文件                    |
| 库文件     | `/opt/<软件名>/<版本>/lib/`（如有）    | 动态库文件（.so）                   |
| 临时文件   | `/data/<软件名>/<版本>/tmp/`           | 临时使用的数据文件                  |
| 环境变量   | `/etc/profile.d/<软件名>.sh`           | 配置环境变量                        |

多版本切换

```bash
ln -sfn /opt/<软件名>/<版本> /opt/nginx/current
ln -sfn /data/<软件名>/<版本> /data/nginx/current

# 重启服务即可应用新版本
sudo systemctl restart nginx
```

> -s：创建软连接
>
> -f：强制执行，如果目标链接已存在则先删除它
>
> -n：当目标是一个符号链接时，不解引用它，防止将目录链接解开，避免报错或错误操作

环境变量

```bash
echo 'export PATH=/opt/<软件名>/current/bin:$PATH' >> /etc/profile.d/<软件名>.sh

chmod +x /etc/profile.d/<软件名>.sh

source /etc/profile.d/<软件名>.sh
```

# 集群分发脚本

```bash
vi /usr/bin/xsync
```

```bash
#!/bin/bash

#1. 判断参数个数
if [ $# -lt 1 ]
then
    echo Not Enough Arguement!
    exit;
fi

#2. 遍历集群所有机器
for host in hadoop000 hadoop001 hadoop002
do
    echo ====================  $host  ====================
    #3. 遍历所有目录，挨个发送

    for file in $@
    do
        #4. 判断文件是否存在
        if [ -e $file ]
            then
                #5. 获取父目录
                pdir=$(cd -P $(dirname $file); pwd)

                #6. 获取当前文件的名称
                fname=$(basename $file)
                ssh $host "mkdir -p $pdir"
                rsync -av $pdir/$fname $host:$pdir
            else
                echo $file does not exists!
        fi
    done
done
```

```bash
# 下载 rsync
yum install -y rsync
# 对脚本授权
chmod +755 /usr/bin/xsync
# 进行集群分发
xsync module/kafka/
```

