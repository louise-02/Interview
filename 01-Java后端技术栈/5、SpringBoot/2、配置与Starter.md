# 2、配置与 Starter

## 1、配置文件

| 格式 | 说明 |
| ---- | ---- |
| `application.yml` / `.properties` | 主配置 |
| `application-{profile}.yml` | 环境：dev、test、prod |
| `bootstrap.yml` | Spring Cloud 早期/bootstrap 上下文，用于连配置中心（Boot 2.4+ 可用 `spring.config.import` 替代） |
| 命令行 / 环境变量 | 优先级较高 |

**多环境**：`spring.profiles.active=prod`。

### 配置优先级（Spring Boot 单机）

**原则：后加载、优先级高的来源会覆盖先加载的同名字段。**

| 优先级（高 → 低） | 来源 |
| :---------------- | :--- |
| 1 | 命令行参数 `--server.port=9090` |
| 2 | Java 系统属性 `-Dserver.port=9090` |
| 3 | 操作系统环境变量 `SERVER_PORT=9090` |
| 4 | jar **外** `./config/application-{profile}.yml` |
| 5 | jar **外** `./config/application.yml` |
| 6 | jar **外** 当前目录 `./application.yml` |
| 7 | jar **内** `classpath:config/application-{profile}.yml` |
| 8 | jar **内** `classpath:application-{profile}.yml` |
| 9 | jar **内** `classpath:application.yml` |
| 10 | `@PropertySource`、默认属性等 |

简化记：**命令行 > 环境变量 > jar 外配置 > jar 内配置**。

### bootstrap.yml 与 application.yml

| | bootstrap.yml | application.yml |
| -- | ------------- | ----------------- |
| 加载时机 | bootstrap 上下文（更早） | 主 application 上下文 |
| 典型用途 | 配置中心地址、应用名、拉远程配置 | 业务配置、数据源、Redis 等 |
| 关系 | 先加载；**同 key 时 application 会覆盖 bootstrap**（主上下文优先级更高） | 后加载，本地默认配置 |

> 老项目常说「bootstrap 优先级高于 application」，指的是 **bootstrap 阶段先连上配置中心、远程配置优先生效**；与本地文件对比时，实际是 **主上下文里的 application.yml 可覆盖 bootstrap 里的同名项**（还受 Cloud 覆盖开关影响）。

### 接入 Spring Cloud 配置中心后

远程配置在 bootstrap / `spring.config.import` 阶段注入，**默认远程配置优先于本地 application.yml**。

| 优先级（高 → 低） | 来源 |
| :---------------- | :--- |
| 1 | 配置中心（Nacos / Config Server 等） |
| 2 | 命令行参数 |
| 3 | 环境变量 / `-D` 系统属性 |
| 4 | 本地 `application.yml` / `application-{profile}.yml` |
| 5 | 本地 `bootstrap.yml` |

Boot 2.4+ 推荐：

```yaml
# application.yml
spring:
  config:
    import: optional:nacos:order-service.yaml
```

### 配置中心覆盖开关（Spring Cloud Config Client）

控制**本地配置能否覆盖远程**：

```yaml
spring:
  cloud:
    config:
      allow-override: true              # 允许本地覆盖远程（默认 true）
      override-none: true               # true：远程配置不可被本地覆盖
      override-system-properties: false # false：系统属性也不覆盖远程
```

| 属性 | 含义 |
| ---- | ---- |
| `allow-override` | 是否允许本地 property 覆盖远程 |
| `override-none` | `true` 时远程配置**不能被**本地覆盖 |
| `override-system-properties` | 是否允许 `-D` / 系统属性覆盖远程 |

Nacos 等客户端另有 `spring.cloud.nacos.config.override-none` 等，以所用组件文档为准。

## 2、配置绑定

```java
@ConfigurationProperties(prefix = "app.order")
public class OrderProperties {
    private int timeout = 30;
}
```

配合 `@EnableConfigurationProperties` 或 `@ConfigurationPropertiesScan`。

**松散绑定**：`app.order-timeout` ↔ `orderTimeout`。

## 3、Starter 机制

`spring-boot-starter-*` = **依赖传递 + 自动配置**。

| Starter | 作用 |
| ------- | ---- |
| spring-boot-starter-web | Web + Tomcat + Jackson |
| spring-boot-starter-data-redis | Redis + Lettuce |
| spring-boot-starter-validation | Hibernate Validator |
| spring-boot-starter-test | JUnit5 + MockMvc |

## 4、自定义 Starter（步骤）

1. 新建模块 `xxx-spring-boot-starter`。
2. 编写 `XxxProperties` + `@ConfigurationProperties`。
3. 编写 `XxxAutoConfiguration` + `@ConditionalOn*`。
4. `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 注册。
5. 业务项目引入 starter 依赖即可。

## 5、日志

默认 **Logback**；`logging.level.root=INFO`；输出 JSON 可换 logstash encoder（SpringBoot.md 有 logback-spring.xml 示例）。

## 6、常见面试题

**1. application.yml 和 bootstrap.yml？**

Cloud 早期用 bootstrap 连配置中心；Boot 2.4+ 推荐 `spring.config.import` 替代 bootstrap。

**2. 如何覆盖自动配置的 Bean？**

自己定义同类型 `@Bean`，`@ConditionalOnMissingBean` 会让自动配置让路。

**3. Starter 和普通 Maven 依赖区别？**

Starter 是 curated 依赖集合 + 对应 AutoConfiguration，不是新功能本身。
