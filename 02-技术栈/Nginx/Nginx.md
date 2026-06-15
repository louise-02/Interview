# Nginx 知识体系

官网：[nginx.org](https://nginx.org) | 文档：[nginx.org/en/docs](https://nginx.org/en/docs/)

Nginx 是高性能 **Web 服务器**、**反向代理**和**负载均衡**组件，常用于静态资源托管、API 网关入口、HTTPS 终结、流量分发。

> **部署实操**（yum / 编译 / Docker / systemd）见 [Deploy/4、nginx.md](../../05-运维和部署/Deploy/4、nginx.md)。

---

## 文档目录

| 序号 | 文件 | 内容 |
|------|------|------|
| 1 | [1、Nginx 简介与架构.md](./1、Nginx%20简介与架构.md) | 核心概念、Master-Worker、请求处理流程、与 Apache 对比 |
| 2 | [2、安装与目录规划.md](./2、安装与目录规划.md) | yum / 编译 / Docker 对比、目录规范、启停与 reload |
| 3 | [3、配置文件详解.md](./3、配置文件详解.md) | 配置层次、main/events/http 块、全局参数逐行说明 |
| 4 | [4、反向代理与负载均衡.md](./4、反向代理与负载均衡.md) | proxy_pass、upstream、负载策略、请求头透传、WebSocket |
| 5 | [5、HTTPS 静态资源与 Rewrite.md](./5、HTTPS%20静态资源与%20Rewrite.md) | SSL/TLS、location 匹配、try_files、SPA、缓存 |
| 6 | [6、性能优化与常见问题.md](./6、性能优化与常见问题.md) | 连接数调优、gzip、限流、日志、故障排查、面试题 |
| 7 | [7、生产综合配置模板.md](./7、生产综合配置模板.md) | **终极模板**：HTTPS 版 + HTTP 内网版，全参数注释 + 按内存档位调参 |

---

## 学习路线

```text
1 简介与架构（知道 Nginx 干什么、怎么跑）
    ↓
2 安装与目录（会装、知道配置文件在哪）
    ↓
3 配置文件详解（读懂 nginx.conf 每一块）
    ↓
4 反向代理与负载均衡（最常用：/api 转发后端）
    ↓
5 HTTPS / 静态 / Rewrite（上线必备）
    ↓
6 性能优化与面试（调 worker、排 502/504）
    ↓
7 生产综合配置模板（按内存选档位，粘贴改 IP 上线）
```

## 常用端口

| 端口 | 说明 |
|------|------|
| 80 | HTTP |
| 443 | HTTPS |

## 与 Spring Cloud Gateway 的分工

| 组件 | 典型职责 |
|------|----------|
| **Nginx** | 边缘 TLS、静态资源、七层反向代理、负载均衡 |
| **Gateway** | 业务路由、统一鉴权、灰度、与 Nacos 等注册中心集成 |

常见链路：**Nginx → Gateway → 微服务**。
