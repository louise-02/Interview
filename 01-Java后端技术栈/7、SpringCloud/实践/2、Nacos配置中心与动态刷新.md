# 2、Nacos 配置中心与动态刷新

在 [1、基础微服务](./1、Nacos+Gateway+OpenFeign搭建微服务.md) 之上，把**可变的业务参数**从本地 `application.yml` 抽到 **Nacos 配置中心**，并演示改配置**不重启**生效。

## 本篇要达成什么

| 步骤 | 结果 |
| ---- | ---- |
| Nacos 控制台新建配置 | `common.yaml`、`order-service.yaml` |
| 各服务接入 config | 启动时拉取远程配置 |
| `@RefreshScope` 演示 | 改 Nacos 折扣率，接口返回值即时变化 |

**知识点**：dataId / group / namespace 规则、配置优先级、`@RefreshScope` 刷新范围。原理见 [2、注册配置与远程调用](../2、注册配置与远程调用.md#2nacos-配置中心)。

---

## 0、前提

- 已完成 [实践/1](./1、Nacos+Gateway+OpenFeign搭建微服务.md)，三个服务能正常注册、Feign、Gateway 路由。
- Nacos 已启动（`8848` / `9848` 可达）。

---

## 1、Maven：加配置中心依赖

**order-service**（本篇重点演示动态刷新）和 **user-service**、**gateway-service** 都加上：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

Boot 2.4+ 推荐用 `spring.config.import` 拉配置，**不必**再引 `spring-cloud-starter-bootstrap`（老项目 bootstrap 方式见文末附录）。

---

## 2、Nacos 控制台：创建配置

登录 `http://127.0.0.1:8848/nacos` → **配置管理 → 配置列表 → 创建配置**。

### 2.1 公共配置 `common.yaml`

| 字段 | 值 |
| ---- | -- |
| Data ID | `common.yaml` |
| Group | `DEFAULT_GROUP` |
| 格式 | YAML |

内容：

```yaml
# 所有服务共享的开关、常量
app:
  env: dev
  feature:
    log-detail: true
```

> **知识点**：公共配置适合放环境标识、功能开关、日志级别等；敏感信息生产环境应加密或使用 Secret。

### 2.2 订单服务专属 `order-service.yaml`

| 字段 | 值 |
| ---- | -- |
| Data ID | `order-service.yaml` |
| Group | `DEFAULT_GROUP` |
| 格式 | YAML |

内容：

```yaml
order:
  discount: 0.9          # 订单折扣，后面改这个做动态刷新演示
  default-shipping: 10   # 默认运费（元）
```

> **知识点 — dataId 默认规则**：未指定时，默认 dataId = `${spring.application.name}.${file-extension}`，即 `order-service.yaml`。也可以显式在 `spring.config.import` 里写其它 dataId。

---

## 3、order-service：接入配置

### 3.1 application.yml

**本地只留端口、服务名、Nacos 地址**；业务参数放 Nacos。

```yaml
server:
  port: 8082

spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        # namespace: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  # 多环境时用命名空间 ID
        group: DEFAULT_GROUP
        # 额外加载公共配置（按顺序，后加载的覆盖先加载的）
        extension-configs:
          - data-id: common.yaml
            group: DEFAULT_GROUP
            refresh: true
  config:
    import:
      - optional:nacos:order-service.yaml   # 主配置；optional 表示 Nacos 暂不可用时不阻断启动
```

**配置加载顺序（简化记忆）**

1. `application.yml`（本地）
2. `extension-configs`（如 `common.yaml`）
3. `spring.config.import` 中的 `order-service.yaml`
4. 后加载的 **覆盖** 同名 key

### 3.2 绑定配置类

```java
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Data
@Component
@RefreshScope                    // Nacos 推送变更后，该 Bean 销毁重建，新值生效
@ConfigurationProperties(prefix = "order")
public class OrderProperties {

    private Double discount = 1.0;
    private Double defaultShipping = 0.0;
}
```

> **知识点 — 哪些会刷新？**
>
> - 加了 `@RefreshScope` 的 Bean，或 `@ConfigurationProperties` + `@RefreshScope`
> - 普通 `@Value` 注入且类上有 `@RefreshScope` 也会刷新
> - **不会**自动刷新的：构造器注入后缓存的常量、static 字段、未标注的 Bean

### 3.3 Controller 使用配置

改造 [实践/1](./1、Nacos+Gateway+OpenFeign搭建微服务.md) 中的 `OrderController`：

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired
    private UserClient userClient;

    @Autowired
    private OrderProperties orderProperties;

    @GetMapping("/{id}")
    public Map<String, Object> getOrder(@PathVariable Long id) {
        double amount = 99.9 * orderProperties.getDiscount()
                + orderProperties.getDefaultShipping();

        Map<String, Object> order = new HashMap<>();
        order.put("orderId", id);
        order.put("amount", amount);
        order.put("discount", orderProperties.getDiscount());
        order.put("shipping", orderProperties.getDefaultShipping());
        order.put("user", userClient.getUser(1L));
        return order;
    }
}
```

---

## 4、user-service / gateway-service（仅接入，不演示刷新）

### user-service application.yml 增量

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        extension-configs:
          - data-id: common.yaml
            group: DEFAULT_GROUP
            refresh: true
  config:
    import:
      - optional:nacos:user-service.yaml
```

Nacos 新建 `user-service.yaml`（可为空或占位）：

```yaml
user:
  default-nickname: 访客
```

### gateway-service application.yml 增量

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
  config:
    import:
      - optional:nacos:gateway-service.yaml
```

`gateway-service.yaml` 示例（路由也可放 Nacos，改完需刷新 Gateway 的 RouteDefinition）：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-route
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
        - id: order-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
```

> Gateway 路由放 Nacos 时，需配合 `@RefreshScope` 或监听 `RefreshRoutesEvent`；入门阶段路由仍放本地 `application.yml` 更简单，远程只放超时等参数即可。

---

## 5、启动与验证

### 5.1 启动

```bash
# 顺序：Nacos → user → order → gateway
mvn -pl user-service spring-boot:run
mvn -pl order-service spring-boot:run
mvn -pl gateway-service spring-boot:run
```

启动日志应出现类似：

```text
Located property source: [BootstrapPropertySource {name='bootstrapProperties-order-service.yaml'}]
```

### 5.2 查接口

```bash
curl http://127.0.0.1:9000/api/orders/100
```

期望（discount=0.9, shipping=10）：

```json
{
  "orderId": 100,
  "amount": 99.91,
  "discount": 0.9,
  "shipping": 10.0,
  "user": { "id": 1, "name": "张三" }
}
```

### 5.3 动态刷新（不重启）

1. Nacos 控制台编辑 `order-service.yaml`，把 `order.discount` 改为 `0.5`，发布。
2. 等待约 **1～3 秒**（Nacos 长轮询推送）。
3. 再次 `curl`，`discount` 应为 `0.5`，`amount` 随之变化。

也可手动触发（需引入 `spring-boot-starter-actuator` 且暴露端点）：

```bash
curl -X POST http://127.0.0.1:8082/actuator/refresh
```

---

## 6、多环境隔离（dev / test / prod）

| 手段 | 用法 |
| ---- | ---- |
| **namespace** | Nacos 控制台建命名空间，复制一套 dataId；客户端 `spring.cloud.nacos.config.namespace=命名空间ID` |
| **group** | 同一 namespace 下用 Group 区分项目或团队 |
| **profile** | dataId 写成 `order-service-dev.yaml`，import 时 `nacos:order-service-${spring.profiles.active}.yaml` |

示例：

```yaml
spring:
  profiles:
    active: dev
  config:
    import:
      - optional:nacos:order-service-${spring.profiles.active}.yaml
```

---

## 7、常见问题

| 现象 | 原因与处理 |
| ---- | ---------- |
| 启动报 `dataId` 找不到 | 检查 Nacos 是否已建同名 yaml；或 import 加 `optional:` |
| 改了 Nacos 不生效 | Bean 未加 `@RefreshScope`；改的是错的 dataId / namespace |
| 本地 yml 与 Nacos 冲突 | 后加载的覆盖前者；把业务 key 只放一处 |
| Gateway 路由不刷新 | 路由 Bean 默认不刷新；用 Actuator `/actuator/gateway/refresh` 或自定义监听 |

---

## 附录：bootstrap 方式（老项目）

Spring Cloud 2020 之前常用 `bootstrap.yml` + `spring-cloud-starter-bootstrap`：

```yaml
# bootstrap.yml（优先级高于 application.yml）
spring:
  application:
    name: order-service
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
```

新项目 **Boot 2.4+** 优先用本篇的 `spring.config.import=nacos:` 写法。

---

**下一篇**：[3、Gateway 鉴权与 Feign 透传 Token](./3、Gateway鉴权与Feign透传Token.md)
