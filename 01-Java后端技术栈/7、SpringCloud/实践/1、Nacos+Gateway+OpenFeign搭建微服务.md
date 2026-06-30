# 1、Nacos + Gateway + OpenFeign 搭建微服务

Nacos 注册发现 + Gateway 统一入口 + OpenFeign 服务间调用。工程按 **`xxx-api` + `xxx-service`** 拆分：**api 只放契约 + DTO**；**Feign 写在调用方 service**（`extends UserApi`）；provider 的 service 实现同一套 `UserApi`。

| 模块 | 端口 | 职责 |
| ---- | ---- | ---- |
| `user-api` | — | `UserApi`、`UserDTO`（纯契约，不引 OpenFeign） |
| `user-service` | 8081 | 实现 `UserApi` |
| `order-api` | — | `OrderApi`、`OrderDTO` |
| `order-service` | 8082 | 实现 `OrderApi`；`UserFeignClient extends UserApi` |
| `gateway-service` | 9000 | 对外入口 |

**Nacos 2.x**：需 JDK 17+；客户端除 8848 外还要能访问 **9848**（gRPC）。默认账号 `nacos` / `nacos`。

---

## 1、前置：部署并启动 Nacos

下面四种方式任选一种，后续 `server-addr: 127.0.0.1:8848` 不变。

| 方式 | 适用 | 数据持久化 |
| ---- | ---- | ---------- |
| Docker 单机 Derby | **本地联调最快** | 否 |
| 二进制单机 Derby | 本机不用 Docker | 否 |
| **单机 + MySQL** | 配置要落库 | 是 |
| **集群 + MySQL** | 生产高可用 | 是 |

### 下载 / 获取 Nacos

| 方式 | 从哪里下 | 说明 |
| ---- | -------- | ---- |
| **二进制包** | [GitHub Releases](https://github.com/alibaba/nacos/releases) | `nacos-server-2.x.x.tar.gz`（示例 2.4.3） |
| 官网 | [nacos.io 下载页](https://nacos.io/download/nacos-server/) | 与 GitHub 同源 |
| **Docker 镜像** | [nacos/nacos-server](https://hub.docker.com/r/nacos/nacos-server) | compose 自动拉取 |
| SQL 脚本 | 包内 `conf/mysql-schema.sql` | 或 GitHub 同版本路径 |

```bash
cd /opt/src
wget https://github.com/alibaba/nacos/releases/download/2.4.3/nacos-server-2.4.3.tar.gz
tar -zxvf nacos-server-2.4.3.tar.gz -C /opt/nacos --strip-components=1
```

### 1.1 Docker 单机（Derby）

仓库内 compose：`05-运维和部署/Deploy/docker/nacos.yml`

```bash
cd /path/to/Deploy/docker
docker compose -f nacos.yml up -d
docker logs -f nacos
```

浏览器：`http://127.0.0.1:8848/nacos` → **服务管理 → 服务列表**。

### 1.2 二进制单机（Derby）

```bash
cd /opt/nacos/bin
sh startup.sh -m standalone
tail -f ../logs/start.out
```

### 1.3 单机 + MySQL

```sql
CREATE DATABASE nacos_config CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nacos'@'%' IDENTIFIED BY 'nacos123';
GRANT ALL PRIVILEGES ON nacos_config.* TO 'nacos'@'%';
FLUSH PRIVILEGES;
```

```bash
wget https://raw.githubusercontent.com/alibaba/nacos/2.4.3/distribution/conf/mysql-schema.sql
mysql -unacos -p nacos_config < mysql-schema.sql
```

`conf/application.properties`：

```properties
nacos.standalone=true
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false&allowPublicKeyRetrieval=true
db.user.0=nacos
db.password.0=nacos123
```

```bash
cd /opt/nacos/bin && sh startup.sh -m standalone
```

Docker + MySQL：`Deploy/docker/nacos-mysql.yml`，改好 `MYSQL_SERVICE_*` 后 `docker compose -f nacos-mysql.yml up -d`。

### 1.4 集群 + MySQL

至少 3 节点，共用同一 `nacos_config` 库；`cluster.conf` 写各节点 `IP:8848`；`application.properties` 配 `nacos.inetutils.ip-address` 与 MySQL；启动 **不要** `-m standalone`：

```bash
sh startup.sh
```

客户端：

```yaml
spring.cloud.nacos.discovery.server-addr: 192.168.1.101:8848,192.168.1.102:8848,192.168.1.103:8848
```

### 1.5 检查

```bash
curl http://127.0.0.1:8848/nacos/v1/console/health/readiness
```

---

## 2、工程结构

```
microservice-demo/
├── pom.xml
├── user-api/                 # UserApi + UserDTO
├── user-service/             # 实现 UserApi
├── order-api/
├── order-service/            # UserFeignClient + 实现 OrderApi
└── gateway-service/
```

**依赖方向**（不要反）：

```text
order-service  →  user-api          ✅  只依赖对方的 api jar
order-service  →  user-service      ❌  不要依赖别人的 service 模块
```

---

## 3、父 POM

Boot **2.7.18** + Cloud **2021.0.9** + Alibaba **2021.0.6.0**（与 Boot 3 混用需整体换 BOM）。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>microservice-demo</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>user-api</module>
        <module>user-service</module>
        <module>order-api</module>
        <module>order-service</module>
        <module>gateway-service</module>
    </modules>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.18</version>
    </parent>

    <properties>
        <java.version>11</java.version>
        <spring-cloud.version>2021.0.9</spring-cloud.version>
        <spring-cloud-alibaba.version>2021.0.6.0</spring-cloud-alibaba.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>user-api</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>order-api</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

---

## 4、user-api（契约 + DTO）

### pom.xml

```xml
<parent>
    <groupId>com.example</groupId>
    <artifactId>microservice-demo</artifactId>
    <version>1.0.0</version>
</parent>
<artifactId>user-api</artifactId>

<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
    </dependency>
</dependencies>
```

只依赖 `spring-web`（提供 `@GetMapping` 等），**不引 OpenFeign**。

### UserDTO

```java
package com.example.user.api.dto;

public class UserDTO {
    private Long id;
    private String name;

    public UserDTO() { }

    public UserDTO(Long id, String name) {
        this.id = id;
        this.name = name;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

### UserApi（provider 实现、consumer 的 Feign 继承）

```java
package com.example.user.api;

import com.example.user.api.dto.UserDTO;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

@RequestMapping("/users")
public interface UserApi {

    @GetMapping("/{id}")
    UserDTO getById(@PathVariable("id") Long id);
}
```

---

## 5、user-service（实现 UserApi）

### pom.xml

```xml
<artifactId>user-service</artifactId>

<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>user-api</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>
```

### application.yml

```yaml
server:
  port: 8081

spring:
  application:
    name: user-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
```

### 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

### UserController

```java
@RestController
public class UserController implements UserApi {

    @Override
    public UserDTO getById(@PathVariable Long id) {
        return new UserDTO(id, "张三");
    }
}
```

---

## 6、order-api

### pom.xml

同 user-api，只依赖 `spring-web`（不需要 openfeign）。

### OrderDTO

```java
package com.example.order.api.dto;

import com.example.user.api.dto.UserDTO;

public class OrderDTO {
    private Long orderId;
    private Double amount;
    private UserDTO user;

    // getter / setter
}
```

### OrderApi

```java
package com.example.order.api;

import com.example.order.api.dto.OrderDTO;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

@RequestMapping("/orders")
public interface OrderApi {

    @GetMapping("/{id}")
    OrderDTO getById(@PathVariable("id") Long id);
}
```

---

## 7、order-service（Feign 调用 user）

### pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>order-api</artifactId>
    </dependency>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>user-api</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
</dependencies>
```

> Cloud 2020+ 无 Ribbon，Feign 必须带 **LoadBalancer**。

### application.yml

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

feign:
  client:
    config:
      default:
        connectTimeout: 3000
        readTimeout: 5000
```

### 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients(basePackages = "com.example.order.feign")
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

### UserFeignClient（写在 order-service，继承 user-api 的 UserApi）

`name` 必须与 Nacos 上 **`user-service` 的 spring.application.name** 一致：

```java
package com.example.order.feign;

import com.example.user.api.UserApi;
import org.springframework.cloud.openfeign.FeignClient;

@FeignClient(name = "user-service", contextId = "orderUserFeignClient")
public interface UserFeignClient extends UserApi {
}
```

> 多个服务都要调 user 时，各自 service 里写自己的 `XxxUserFeignClient extends UserApi`（或抽 `user-feign-client` 模块复用）。Sentinel **fallback** 也加在这个接口上，见实践 4。

### OrderController

```java
@RestController
public class OrderController implements OrderApi {

    @Resource
    private UserFeignClient userFeignClient;

    @Override
    public OrderDTO getById(@PathVariable Long id) {
        OrderDTO order = new OrderDTO();
        order.setOrderId(id);
        order.setAmount(99.9);
        order.setUser(userFeignClient.getById(1L));
        return order;
    }
}
```

---

## 8、gateway-service

### pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
</dependencies>
```

不要引入 `spring-boot-starter-web`（与 Gateway WebFlux 冲突）。

### application.yml

```yaml
server:
  port: 9000

spring:
  application:
    name: gateway-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
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

| 外部请求 | Gateway 转发后 | 落到 |
| -------- | -------------- | ---- |
| `GET /api/users/1` | `GET /users/1` | user-service |
| `GET /api/orders/100` | `GET /orders/100` | order-service |

### 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

---

## 9、启动与验证

1. Nacos 已启动  
2. `user-service` → `order-service` → `gateway-service`（Feign 前 user 须已注册）  
3. 控制台看到三个服务各 1 个实例  

```bash
curl http://127.0.0.1:8081/users/1

curl http://127.0.0.1:9000/api/orders/100
```

期望：

```json
{
  "orderId": 100,
  "amount": 99.9,
  "user": { "id": 1, "name": "张三" }
}
```

---

## 10、常见问题

| 现象 | 处理 |
| ---- | ---- |
| 注册不上 Nacos | `server-addr`、9848 是否可达 |
| Feign `Connection refused` | user-service 未注册；`@FeignClient(name)` 须为 `user-service` |
| Feign 404 | `UserApi` 路径与 user-service 实现不一致（改 api 一处即可） |
| Feign 找不到 Bean | `@EnableFeignClients(basePackages = "com.example.order.feign")` |
| Gateway 503 | 目标未注册；缺 LoadBalancer |
| Gateway 与 Web 冲突 | gateway 模块不要 `spring-boot-starter-web` |

