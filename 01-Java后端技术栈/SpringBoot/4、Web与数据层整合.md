# 4、Web 与数据层整合

## 1、spring-boot-starter-web

自动配置：

- **DispatcherServlet**、Jackson HttpMessageConverter
- 内嵌 **Tomcat**（可换 Jetty/Undertow）
- 默认错误页、静态资源路径

## 2、整合 MyBatis-Plus

```xml
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>mybatis-plus-boot-starter</artifactId>
</dependency>
```

`@MapperScan`、配置 mapper-locations、type-aliases-package。分页插件 `PaginationInnerInterceptor`。

## 3、整合 Redis

`spring-boot-starter-data-redis` + Lettuce 连接池。

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      lettuce:
        pool:
          max-active: 8
```

RedisTemplate 序列化：Key StringRedisSerializer，Value JSON 或 String。缓存 `@EnableCaching` + `@Cacheable`。

## 4、整合 RabbitMQ / Kafka

`spring-boot-starter-amqp`：`@RabbitListener`；

`spring-kafka`：`@KafkaListener`，配置 bootstrap-servers、group-id、ack 模式。

## 5、整合 Validation

`spring-boot-starter-validation`（Hibernate Validator），Controller `@Valid`，统一异常处理 MethodArgumentNotValidException。

## 6、整合 OpenAPI

springdoc-openapi / Knife4j：`@Operation`、`@Schema`，生成 Swagger UI。

## 7、多环境配置实践

```yaml
# application.yml
spring:
  profiles:
    active: @profiles.active@  # Maven 过滤
---
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dev
```

敏感信息环境变量注入，不进 Git。

## 8、CORS 与 WebMvcConfigurer

addCorsMappings、addInterceptors、addResourceHandlers 集中 Web 配置（SpringBoot.md 有拦截器代码示例）。

## 9、异步 @Async

`@EnableAsync`，线程池 `TaskExecutor` Bean 自定义 core/max/queue，注意异常处理和 ThreadLocal 传递。

## 10、Scheduling

`@EnableScheduling` + `@Scheduled(cron/fixedDelay)`，分布式需 ShedLock 或 XXL-JOB 避免重复执行。
