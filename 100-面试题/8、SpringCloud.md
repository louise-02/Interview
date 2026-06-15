# SpringCloud 面试题

本文件可**单独阅读**，每题含完整参考答案。

---

### 1. 什么是微服务？优缺点？

将单体拆成多个**独立部署**的小服务，按业务能力边界划分，独立扩缩、技术栈可选。优点：团队自治、故障隔离、弹性。缺点：分布式复杂度、运维成本、数据一致性、链路排查难。

---

### 2. Spring Cloud 是什么？

在 Spring Boot 之上的**微服务工具集**，不是新运行时。提供注册发现、配置中心、负载均衡、网关、熔断限流、分布式事务、链路追踪等能力，常用 **Spring Cloud Alibaba**（Nacos、Sentinel、Seata）或 Netflix 组件（部分已停更）。

---

### 3. 注册中心作用？Nacos 做什么？

服务实例启动时**注册** IP、端口、元数据；消费者**订阅**服务名获取健康实例列表。Nacos 同时做**配置中心**，支持 namespace/group 隔离环境，临时实例 AP、持久实例可 CP。

---

### 4. Eureka 和 Nacos 区别？

Eureka：Netflix AP 架构，**停更**，仅注册。Nacos：注册+配置一体，支持 AP/CP 切换，国内主流，阿里维护。

---

### 5. OpenFeign 原理？

声明式 HTTP 客户端：接口 + 注解 → JDK **动态代理** → 解析 @GetMapping 等 → **LoadBalancer** 选实例 → HTTP 客户端（HttpURLConnection/HttpClient/OkHttp）发请求。可配超时、重试、请求拦截器传 Token。

---

### 6. Ribbon 和 LoadBalancer？

Ribbon 客户端负载均衡**停更**。Spring Cloud **LoadBalancer** 替代，配合 `@LoadBalanced RestTemplate` 或 Feign 内置，从注册中心拿实例列表，策略 RoundRobin 等。

---

### 7. 为什么 Feign 第一次调用慢？

懒加载 Feign Client、连接池未预热、DNS 解析。可启动 warmup、配置 httpclient 连接池 maxTotal、合理超时。

---

### 8. Spring Cloud Gateway 是什么？

基于 **WebFlux** 的反应式 **API 网关**，路由 Predicate + Filter。支持 lb://service-name 负载均衡、限流、鉴权、灰度。与 Zuul1（Servlet 阻塞）不同，Gateway 是非阻塞模型。

---

### 9. Gateway 和 Nginx 分工？

Nginx：边缘 TLS、静态资源、四层/七层反向代理。Gateway：**业务路由**、统一鉴权、聚合 API、灰度、与注册中心集成。常 Nginx → Gateway → 微服务。

---

### 10. Sentinel 做什么？

**流量控制**（QPS/并发线程数）、**熔断降级**（慢调用比例、异常比例）、系统自适应保护。规则可推送到控制台，持久化 Nacos。Feign 整合 fallback 返回降级数据。

---

### 11. 熔断和限流区别？

**限流**：防止流量过大打垮系统，超阈值快速失败。**熔断**：依赖**故障**时打开断路器，一段时间 fail-fast，半开试探恢复，防**雪崩**。

---

### 12. 什么是服务雪崩？

下游慢/挂 → 上游线程阻塞等待 → 线程池满 → 上游也不可用 → 连锁扩散。解决：超时、熔断、限流、隔离（舱壁）、降级。

---

### 13. Hystrix 还能用吗？

**停更**，维护模式。新项目用 **Sentinel** 或 Resilience4j。

---

### 14. Seata AT 模式原理？

**AT 自动补偿**：一阶段业务 SQL 正常执行，Seata 解析 SQL 生成 **undo log** 并提交本地事务；二阶段 TC 通知 **commit** 删 undo 或 **rollback** 用 undo 恢复。需数据源代理，适合多数 CRUD。

---

### 15. Seata TCC 和 AT 选型？

AT 无侵入、开发快，需支持的数据库代理。TCC 手写 Try/Confirm/Cancel，性能好、控制强，适合高价值交易。能最终一致则用 **消息队列** 往往更简单。

---

### 16. 分布式事务一定需要 Seata吗？

不一定。优先：**本地事务** + **最终一致**（本地消息表、MQ 事务消息、Canal 订阅 binlog）。跨库强一致代价高，能避免则避免。

---

### 17. 配置中心动态刷新？

Nacos 改配置 → 推送客户端 → @RefreshScope Bean 销毁重建 → @ConfigurationProperties 重新绑定。Feign 客户端 URL 等也可刷新。

---

### 18. CAP 怎么理解？

分区 P 发生时，只能在 **C 一致性** 和 **A 可用性** 间权衡。注册中心临时实例偏 AP；配置/主从选 CP 场景用持久化模式。

---

### 19. 如何做灰度发布？

Gateway 按 Header/权重路由到不同版本服务；K8s 多 Deployment + Service 权重；Nacos 元数据 version 过滤实例。

---

### 20. 链路追踪 TraceId？

网关或 Sleuth 生成 TraceId/SpanId，Feign/RestTemplate 拦截器传递 Header，Zipkin/SkyWalking 收集，排查跨服务慢调用。

---

### 21. Feign 超时和重试配置？

feign.client.config.default.connectTimeout/readTimeout。Retryer 默认可能重试，**非幂等接口**（下单）应关闭重试或仅 GET 重试。

---

### 22. 服务间认证？

JWT 在 Gateway 校验，下游信任内网或再验签；或 mTLS 服务网格；避免裸 RPC 公网暴露。

---

### 23. 注册中心挂了怎么办？

客户端有**本地缓存**实例列表，短期可继续调用；新实例无法注册、无法发现新节点，需集群高可用部署 Nacos。

---

### 24. Spring Cloud 和 Dubbo 区别？

Cloud 以 **HTTP REST** + 注册中心为主；Dubbo **RPC**（TCP、hessian2/protobuf），性能、泛化调用、服务治理模型不同。可混用（Dubbo + Nacos）。

---

### 25. namespace 和 group 在 Nacos？

**namespace** 通常隔离环境（dev/test/prod）。**group** 同环境下分组（如 DEFAULT_GROUP 或业务组）。dataId 为配置文件名。

---

### 26. 如何保证接口幂等？

Token 表、Redis setnx 幂等键、数据库唯一索引、状态机 CAS。消费端 MQ **幂等**同样重要。

---

### 27. 分布式 ID 方案？

雪花 Snowflake、号段模式、UUID（无序不适合索引）、Redis INCR。时钟回拨是雪花经典问题，需处理。

---

### 28. 配置和注册能否分开？

可以。Eureka+Apollo、Consul 二合一、或 Nacos 两者都做。分离则注册挂不影响配置读取，职责清晰。

---

### 29. Spring Cloud Bus 做什么？

基于 MQ 广播配置变更事件，刷新所有实例（与 Bus AMQP/Kafka 配合）。现多用 Nacos 长轮询/推送替代部分场景。

---

### 30. 微服务如何划分？

按**业务能力**（订单、用户、支付），高内聚低耦合，避免分布式单体（过细调用链）。数据归属单一团队（康威定律）。

---

### 31. 版本兼容性注意？

Spring Cloud **Release Train** 与 Boot 版本严格对应，如 2021.0.x 配 Boot 2.6/2.7，2022.x 配 Boot 3。Alibaba 版本三者对齐查官方表格。

---

### 32. 如何做全链路压测？

影子库/影子 topic、流量染色 Header、压测账号隔离，避免污染生产数据。
