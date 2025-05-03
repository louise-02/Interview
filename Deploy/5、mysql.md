# 1、docker 安装

1.1、更改镜像源

创建或修改配置文件

```bash
sudo vi /etc/docker/daemon.json
```

添加国内镜像源地址

```bash
{
  "registry-mirrors": [
    "https://registry.docker-cn.com",          // Docker 中国官方镜像（推荐）
    "https://mirror.ccs.tencentyun.com",       // 腾讯云镜像
    "https://docker.mirrors.ustc.edu.cn",      // 中科大镜像
    "https://hub-mirror.c.163.com",            // 网易云镜像
    "https://<你的ID>.mirror.aliyuncs.com"     // 阿里云镜像（需注册后获取）
  ]
}
```

重启 Docker 服务

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

验证配置是否生效

```bash
docker info

Registry Mirrors:
  https://registry.docker-cn.com/
  https://mirror.ccs.tencentyun.com/
```

1.2、拉取镜像

1.3、启动