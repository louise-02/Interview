# 4、Sentinel 限流熔断与 Nacos 持久化

在 [1、基础微服务](./1、Nacos+Gateway+OpenFeign搭建微服务.md) 之上接入 **Sentinel**：

- **Gateway 网关限流**（按路由 QPS）
- **Feign 调用熔断降级**（user 慢/挂时 order 返回 fallback）
- **规则持久化到 Nacos**（重启不丢规则）

原理见 [3、网关限流与分布式事务](../3、网关限流与分布式事务.md#2sentinel)。

---

## 本篇要达成什么

| 场景 | 规则 | 预期 |
| ---- | ---- | ---- |
| 狂刷 `GET /api/users/1` | Gateway QPS = 2 | 超出返回 429 或 Sentinel 默认 blocked |
| user-service 睡眠 3s | Feign 慢调用熔断 | order 走 fallback，不长时间阻塞 |
| 重启 gateway | Nacos 中 gw-flow 规则 | 限流仍生效 |

---

## 0、前提

- 基础三服务 + Nacos 已就绪。
- 可选：已完成 [实践/3](./3、Gateway鉴权与Feign透传Token.md)（本篇 curl 不带 Token，若开了鉴权需加 Header）。

---

## 1、启动 Sentinel Dashboard

开发环境 Docker 最快：

```bash
docker run -d --name sentinel \
  -p 8858:8858 \
  blueriver/sentinel-dashboard:1.8.6
```

浏览器 `http://127.0.0.1:8858`，默认账号 **sentinel / sentinel**。

> 生产应用名与 `spring.application.name` 一致才能在控制台看到资源；客户端需配置 `spring.cloud.sentinel.transport.dashboard=127.0.0.1:8858`。

---

## 2、父 POM / 各模块依赖

### gateway-service

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>
```

### order-service

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

### user-service（演示慢调用，可选）

仅加 sleep，不必引 Sentinel。

---

## 3、Gateway：限流 + Nacos 持久化

### 3.1 application.yml

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: 127.0.0.1:8858
        port: 8719                    # 与 Dashboard 通信的本地端口
      eager: true                     # 启动即连 Dashboard，否则首次访问才连
      datasource:
        # 网关流控规则 — 存 Nacos，重启自动加载
        gw-flow:
          nacos:
            server-addr: 127.0.0.1:8848
            dataId: sentinel-gateway-gw-flow
            groupId: SENTINEL_GROUP
            rule-type: gw-flow
        # 网关 API 分组（可选，用于更细粒度路由）
        gw-api-group:
          nacos:
            server-addr: 127.0.0.1:8848
            dataId: sentinel-gateway-gw-api-group
            groupId: SENTINEL_GROUP
            rule-type: gw-api-group
```

> **知识点 — rule-type**：`flow` 用于普通资源；Gateway 专用 `gw-flow`、`gw-api-group`、`degrade` 等，dataId 需与 Nacos 中 JSON 对应。

### 3.2 在 Nacos 创建网关流控规则

**配置管理 → 创建配置**

| 字段 | 值 |
| ---- | -- |
| Data ID | `sentinel-gateway-gw-flow` |
| Group | `SENTINEL_GROUP` |
| 格式 | JSON |

内容（限制 **user-route** 资源 QPS = 2）：

```json
[
  {
    "resource": "user-route",
    "resourceMode": 0,
    "grade": 1,
    "count": 2,
    "intervalSec": 1,
    "controlBehavior": 0,
    "burst": 0,
    "maxQueueingTimeoutMs": 500
  }
]
```

字段说明：

| 字段 | 含义 |
| ---- | ---- |
| `resource` | 对应 `spring.cloud.gateway.routes[].id` |
| `grade` | 1=QPS 限流，0=并发线程数 |
| `count` | 阈值 |
| `intervalSec` | 统计窗口（秒） |

创建空的 API 分组（可选）：

Data ID：`sentinel-gateway-gw-api-group`，Group：`SENTINEL_GROUP`，内容 `[]`。

### 3.3 自定义限流响应（可选）

```java
import com.alibaba.csp.sentinel.adapter.gateway.sc.callback.BlockRequestHandler;
import com.alibaba.csp.sentinel.adapter.gateway.sc.callback.GatewayCallbackManager;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.server.ServerResponse;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import javax.annotation.PostConstruct;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class GatewaySentinelConfig {

    @PostConstruct
    public void init() {
        BlockRequestHandler handler = (ServerWebExchange exchange, Throwable t) -> {
            Map<String, Object> body = new HashMap<>();
            body.put("code", 429);
            body.put("msg", "请求过于频繁，请稍后再试");
            return ServerResponse.status(HttpStatus.TOO_MANY_REQUESTS)
                    .contentType(MediaType.APPLICATION_JSON)
                    .body(BodyInserters.fromValue(body));
        };
        GatewayCallbackManager.setBlockHandler(handler);
    }
}
```

### 3.4 验证 Gateway 限流

```bash
# 快速请求 5 次
for i in {1..5}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:9000/api/users/1
done
```

前 2 次左右 `200`，后续出现 `429`（或 Sentinel 默认页）。

Dashboard **网关流控** 菜单可看到 `user-route` 的 pass / block 数。

---

## 4、order-service：Feign + 熔断降级

### 4.1 application.yml

```yaml
feign:
  sentinel:
    enabled: true          # 开启 Feign 整合 Sentinel

spring:
  cloud:
    sentinel:
      transport:
        dashboard: 127.0.0.1:8858
      eager: true
      datasource:
        degrade:
          nacos:
            server-addr: 127.0.0.1:8848
            dataId: sentinel-order-degrade
            groupId: SENTINEL_GROUP
            rule-type: degrade
        flow:
          nacos:
            server-addr: 127.0.0.1:8848
            dataId: sentinel-order-flow
            groupId: SENTINEL_GROUP
            rule-type: flow
```

> Feign 资源名默认：`GET:http://user-service/users/{id}`（方法+URL），Dashboard 里能看到精确名称。

### 4.2 Feign fallbackFactory

```java
import org.springframework.cloud.openfeign.FallbackFactory;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        return id -> {
            Map<String, Object> fallback = new HashMap<>();
            fallback.put("id", id);
            fallback.put("name", "用户服务暂不可用");
            fallback.put("degraded", true);
            fallback.put("reason", cause.getClass().getSimpleName());
            return fallback;
        };
    }
}
```

Feign 接口改为：

```java
@FeignClient(name = "user-service", fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient {

    @GetMapping("/users/{id}")
    Map<String, Object> getUser(@PathVariable("id") Long id);
}
```

> **知识点**：`fallback` 写死返回值；`fallbackFactory` 能拿到异常原因，便于日志与区分超时/404/熔断。

### 4.3 Nacos：降级规则

Data ID：`sentinel-order-degrade`，Group：`SENTINEL_GROUP`，JSON：

```json
[
  {
    "resource": "GET:http://user-service/users/{id}",
    "grade": 0,
    "count": 500,
    "timeWindow": 10,
    "minRequestAmount": 5,
    "statIntervalMs": 1000,
    "slowRatioThreshold": 0.5
  }
]
```

含义（**慢调用比例**熔断）：

- 1 秒内至少 5 次请求
- 慢调用（超过 500ms）比例 ≥ 50%
- 熔断打开 **10 秒**，期间走 fallback

若 user 直接报错，可再加 **异常比例** 规则（`grade: 1`）：

```json
{
  "resource": "GET:http://user-service/users/{id}",
  "grade": 1,
  "count": 0.5,
  "timeWindow": 10,
  "minRequestAmount": 5,
  "statIntervalMs": 1000
}
```

### 4.4 user-service：模拟慢接口

```java
@GetMapping("/{id}")
public Map<String, Object> getById(@PathVariable Long id) throws InterruptedException {
    Thread.sleep(3000);   // 演示慢调用，触发熔断
    Map<String, Object> user = new HashMap<>();
    user.put("id", id);
    user.put("name", "张三");
    return user;
}
```

### 4.5 验证 Feign 熔断

连续访问 order（经 Gateway 或直连 8082）：

```bash
for i in {1..10}; do
  curl -s http://127.0.0.1:9000/api/orders/100 | head -c 120
  echo
done
```

多次慢调用后，`user` 部分出现 `"degraded": true`，说明 fallback 生效。Dashboard **簇点链路** 里可见 Feign 资源与熔断状态。

---

## 5、控制台动态推规则（开发调试用）

1. 启动 gateway / order 并访问几次，让资源出现在 Dashboard。
2. **流控规则 → 新增**，选中 `user-route` 或 Feign 资源。
3. 生产应 **写回 Nacos JSON**，否则重启丢失；控制台改动与 Nacos 双向同步需额外组件，一般 **以 Nacos 为唯一数据源**。

---

## 6、规则类型速查

| rule-type | 用途 | 典型 resource |
| --------- | ---- | ------------- |
| `flow` | QPS / 并发限流 | 接口名、Feign 资源名 |
| `degrade` | 熔断 | 慢调用 / 异常比例 / 异常数 |
| `gw-flow` | Gateway 路由限流 | route id |
| `gw-api-group` | Gateway API 分组 | 自定义 API 名 |
| `param-flow` | 热点参数 | 如商品 ID |

---

## 7、与 Gateway 自带 RequestRateLimiter 对比

| | Sentinel | Gateway RequestRateLimiter |
| -- | -------- | -------------------------- |
| 能力 | 限流 + 熔断 + 系统保护 |  mainly 限流 |
| 规则存储 | Nacos / Dashboard | 通常写 yaml 或 Redis |
| Feign | 原生整合 | 需自行处理 |

项目已用 Alibaba 栈时，**Sentinel 统一治理**更常见。

---

## 8、常见问题

| 现象 | 处理 |
| ---- | ---- |
| Dashboard 看不到应用 | `spring.application.name` 与访问产生流量；`eager: true`；防火墙 8719 |
| 规则不生效 | Nacos dataId/group 与 yml 不一致；JSON 格式错误 |
| Feign 无 fallback | 未 `feign.sentinel.enabled=true`；接口未写 `fallbackFactory` |
| 资源名对不上 | Dashboard 看实际 resource 字符串，Nacos JSON 与之完全一致 |
| 限流 429 但无 JSON body | 配置 `GatewaySentinelConfig` 自定义 BlockHandler |

---

## 9、串联全栈验证清单

按顺序跑通整个实践系列：

| 序号 | 文档 | 验证点 |
| ---- | ---- | ------ |
| 1 | [基础微服务](./1、Nacos+Gateway+OpenFeign搭建微服务.md) | Nacos 三服务注册；Gateway curl order |
| 2 | [Nacos 配置](./2、Nacos配置中心与动态刷新.md) | 改 discount 不重启生效 |
| 3 | [鉴权透传](./3、Gateway鉴权与Feign透传Token.md) | login → Token → order 里 user.requestedBy |
| 4 | 本篇 | QPS 429；user 慢 → fallback |

---

## 10、多实例负载均衡（补充）

同一服务起多个实例，Feign / Gateway 自动 **RoundRobin**：

```bash
# 终端 1
mvn -pl user-service spring-boot:run

# 终端 2（改端口）
mvn -pl user-service spring-boot:run -Dspring-boot.run.arguments=--server.port=8083
```

Nacos 中 `user-service` 应有 **2 个健康实例**。在 `UserController` 返回本机端口，多次 curl 可看到轮询：

```java
@Value("${server.port}")
private String port;

user.put("instance", port);
```

无需额外配置 LoadBalancer（已引 `spring-cloud-starter-loadbalancer`）。
