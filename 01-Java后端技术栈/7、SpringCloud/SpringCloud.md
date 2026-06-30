# Spring Cloud

Spring Cloud 是在 **Spring Boot** 之上的**微服务治理**工具集，提供注册发现、配置、网关、负载均衡、熔断限流、分布式事务等（多由 **Spring Cloud Alibaba** 或 Netflix/其他实现）。

## 知识体系

| 章节 | 内容 |
| ---- | ---- |
| [1、微服务架构与组件选型](./1、微服务架构与组件选型.md) | 拆分原则、CAP、常见组件对照 |
| [2、注册配置与远程调用](./2、注册配置与远程调用.md) | Nacos、OpenFeign、LoadBalancer |
| [3、网关限流与分布式事务](./3、网关限流与分布式事务.md) | Gateway、Sentinel、Seata |
| [4、版本兼容与生产实践](./4、版本兼容与生产实践.md) | 版本对齐、可观测、发布、安全 |

## 开发实践

按顺序做，后一篇在前一篇工程上增量改造：

| 序号 | 文件 | 内容 |
| ---- | ---- | ---- |
| 1 | [Nacos+Gateway+OpenFeign搭建微服务](./实践/1、Nacos+Gateway+OpenFeign搭建微服务.md) | api/service 拆分、UserApi + UserFeignClient、Gateway、验证 |
| 2 | [Nacos配置中心与动态刷新](./实践/2、Nacos配置中心与动态刷新.md) | 远程 yaml、`@RefreshScope`、改配置不重启 |
| 3 | [Gateway鉴权与Feign透传Token](./实践/3、Gateway鉴权与Feign透传Token.md) | JWT 登录、GlobalFilter、RequestInterceptor |
| 4 | [Sentinel限流熔断与Nacos持久化](./实践/4、Sentinel限流熔断与Nacos持久化.md) | Dashboard、网关 QPS、Feign fallback、多实例 |

## 与 Boot 关系

每个微服务是独立 **Spring Boot** 应用；Cloud 提供**跨服务**能力。部署见 [Nacos 文档](../../../05-运维和部署/Deploy/13、nacos.md)。

## 版本

Spring Cloud **2021.x / 2022.x / 2023.x** 与 Boot 2.7 / 3.x 有对应 BOM，需查 [官方兼容表](https://spring.io/projects/spring-cloud)。

Spring Cloud Alibaba 版本与 Cloud、Boot 三者对齐（如 2022.0.0.0 对应 Cloud 2022 + Boot 3）。
