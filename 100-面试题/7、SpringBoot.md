# SpringBoot 面试题

本文件可**单独阅读**，每题含完整参考答案。

---

### 1. Spring Boot 是什么？和 Spring 关系？

Spring Boot 是在 Spring Framework 上的**快速开发框架**，提供：自动配置、Starter 依赖聚合、内嵌 Tomcat/Jetty、生产就绪特性（Actuator）、约定优于配置。每个 Boot 应用仍是 Spring 容器，只是启动和配置大幅简化。

---

### 2. @SpringBootApplication 组成？

等于 @Configuration + @EnableAutoConfiguration + @ComponentScan。main 中 SpringApplication.run 启动内嵌容器并刷新 ApplicationContext。

---

### 3. 自动配置原理？

@EnableAutoConfiguration 导入 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Boot3）或 spring.factories（Boot2）中的配置类。这些类用 @ConditionalOnClass、OnMissingBean、OnProperty 等**条件注解**，满足才注册 Bean。debug 模式可看 Positive/Negative matches。

---

### 4. 如何排除某个自动配置？

@SpringBootApplication(exclude = DataSourceAutoConfiguration.class) 或配置 `spring.autoconfigure.exclude=全限定类名`。

---

### 5. Starter 是什么？

如 spring-boot-starter-web = web 相关依赖 + 对应 AutoConfiguration。自定义 Starter：xxx-autoconfigure 模块 + Properties + @Conditional 配置类 + META-INF 注册 + xxx-starter 只引依赖。

---

### 6. application.yml 加载顺序？

命令行参数 > Java 系统属性 > OS 环境变量 > profile 特定文件 application-{profile}.yml > application.yml。Cloud 时代还可用 spring.config.import 拉 Nacos。

---

### 7. @ConfigurationProperties 和 @Value？

@ConfigurationProperties(prefix="app") 批量绑定、类型安全、支持嵌套、IDE 提示。 @Value 单个 SpEL/占位符，简单字段可用。复杂配置推荐 ConfigurationProperties + 校验 @Validated。

---

### 8. 多环境 Profile？

spring.profiles.active=dev,prod 激活。application-dev.yml 覆盖默认。 @Profile("dev") Bean 按环境注册。

---

### 9. Actuator 有哪些端点？

health（K8s 探针）、info、metrics、env、beans、loggers 等。生产通过 management.endpoints.web.exposure.include 最小暴露，并加 Security 或网关鉴权，**不要**把 env/heapdump 暴露公网。

---

### 10. jar 包为什么能 java -jar 运行？

Spring Boot 打 **fat jar**，MANIFEST Main-Class 为 JarLauncher，加载 BOOT-INF/lib 下依赖和 BOOT-INF/classes，再启动内嵌容器和 main 方法。

---

### 11. 内嵌 Tomcat 如何改端口？

server.port=8080，context-path server.servlet.context-path=/api。

---

### 12. 如何优雅停机？

server.shutdown=graceful，spring.lifecycle.timeout-per-shutdown-phase=30s，K8s preStop + 足够 terminationGracePeriodSeconds，Drain 进行中请求。

---

### 13. DevTools 原理？生产能用吗？

开发 classpath 双 ClassLoader，改 class 快速重启。**生产禁止**引入 devtools 依赖。

---

### 14. Boot 2 升 Boot 3 注意？

JDK **17+** 必须；javax.* 改 **jakarta.***；部分配置项更名；Spring Security 6 规则变化；第三方库需 jakarta 兼容版。

---

### 15. 如何定制嵌入式容器？

实现 WebServerFactoryCustomizer<TomcatServletWebServerFactory> 改线程数、连接数等。

---

### 16. Spring Boot 如何读配置到静态字段？

不推荐 static @Value；用实例 @ConfigurationProperties Bean，静态方法通过注入实例访问，或 @PostConstruct 赋值给 static。

---

### 17. 日志默认用什么？

Logback（spring-boot-starter-logging），可 logging.level.com.example=DEBUG。JSON 日志用 logstash-logback-encoder。

---

### 18. @SpringBootTest 做什么？

集成测试，启动完整或部分上下文。 @WebMvcTest 只测 Controller 层 + MockMvc。 @DataJpaTest 测 JPA。

---

### 19. 为什么自动配置不会覆盖我的 Bean？

AutoConfiguration 里多用 @ConditionalOnMissingBean，你自定义同类型 Bean 后自动配置让路。

---

### 20. spring.factories 和 AutoConfiguration.imports 区别？

Boot 2.x 用 META-INF/spring.factories 键 org.springframework.boot.autoconfigure.EnableAutoConfiguration。Boot 3 改为 **AutoConfiguration.imports** 纯列表文件，更清晰。

---

### 21. 如何实现配置动态刷新？

Spring Cloud @RefreshScope + 配置中心推送；或 EnvironmentChangeEvent 监听。非 Cloud 项目可自研或 Actuator /refresh（需小心安全）。

---

### 22. 打包 war 部署外部 Tomcat？

继承 SpringBootServletInitializer，override configure，打 war 包，部署外部容器。内嵌 jar 更常见。

---

### 23. 如何关闭 Banner？

spring.main.banner-mode=off。

---

### 24. 启动慢怎么优化？

LazyInitializationBeanFactory、排除无用 AutoConfiguration、减少 classpath 扫描范围 @ComponentScan 指定包、AOT 原生镜像（Native）。

---

### 25. Spring Boot 和 Spring Cloud 关系？

Boot 是**单应用**基础；Cloud 是**多服务治理**（注册、配置、网关、熔断），Cloud 构建在 Boot 之上，每个微服务是一个 Boot 应用。

---

### 26. 如何打印生效的配置和 Bean？

--debug 或 logging.level.org.springframework.boot.autoconfigure=DEBUG；Actuator /beans、/conditions（需暴露）。

---

### 27. 自定义 health 指标？

实现 HealthIndicator 或 ReactiveHealthIndicator，返回 Health.up()/down()，合并到 /actuator/health。

---

### 28. 分层 jar（layered jar）好处？

Docker 构建时分层 COPY，依赖层不变则缓存命中，加快镜像构建。

---

### 29. spring.main.allow-circular-references？

Boot 2.6+ 默认禁止循环依赖；legacy 可 true 恢复旧行为，**不推荐**，应重构设计。

---

### 30. 如何集成 MyBatis/Redis？

引入 starter-mybatis、starter-data-redis，写配置，AutoConfiguration 注册 SqlSessionFactory、RedisTemplate。详见各 Starter 文档。
