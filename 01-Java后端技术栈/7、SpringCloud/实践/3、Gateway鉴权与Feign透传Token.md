# 3、Gateway 鉴权与 Feign 透传 Token

在 [1、基础微服务](./1、Nacos+Gateway+OpenFeign搭建微服务.md) 之上，实现：

1. **Gateway 统一鉴权**：无 Token 拒绝；有合法 JWT 才转发。
2. **Feign 透传 Token**：`order-service` 调 `user-service` 时，把上游请求的 `Authorization` 带过去。
3. **下游校验**：`user-service` 校验 Token，区分「网关来的请求」和「Feign 内部调用」。

原理：Gateway 是**南北流量**入口；Feign 是**东西流量**，默认**不会**自动带 Header，需 `RequestInterceptor`。见 [2、注册配置与远程调用](../2、注册配置与远程调用.md#3openfeign)。

---

## 本篇架构

```
客户端 --(JWT)--> Gateway --(转发 Authorization)--> order-service
                                                      |
                                           Feign + RequestInterceptor
                                                      |
                                                      v
                                               user-service（校验 JWT）
```

| 接口 | 是否鉴权 |
| ---- | -------- |
| `POST /api/auth/login` | 白名单，登录拿 Token |
| `GET /api/users/{id}` | 需要 Token |
| `GET /api/orders/{id}` | 需要 Token（内部 Feign 透传） |

---

## 0、前提

- 基础三服务已跑通。
- 建议已读 [实践/2](./2、Nacos配置中心与动态刷新.md)（非必须）。

---

## 1、父 POM：JWT 依赖（供 gateway 使用）

在父 `pom.xml` 的 `<properties>` 加：

```xml
<jjwt.version>0.11.5</jjwt.version>
```

在 `<dependencyManagement>` 加：

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>${jjwt.version}</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>${jjwt.version}</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>${jjwt.version}</version>
</dependency>
```

---

## 2、gateway-service：登录 + 鉴权 Filter

### 2.1 pom.xml 增加

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <scope>runtime</scope>
</dependency>
```

### 2.2 JWT 工具类

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;

public class JwtUtil {

    // 演示用固定密钥；生产必须用足够长的随机密钥并放配置中心
    private static final String SECRET = "microservice-demo-jwt-secret-key-32bytes!";
    private static final SecretKey KEY = Keys.hmacShaKeyFor(SECRET.getBytes(StandardCharsets.UTF_8));
    private static final long EXPIRE_MS = 3600_000;

    public static String generateToken(String userId, String username) {
        return Jwts.builder()
                .setSubject(userId)
                .claim("username", username)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRE_MS))
                .signWith(KEY, SignatureAlgorithm.HS256)
                .compact();
    }

    public static Claims parseToken(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(KEY)
                .build()
                .parseClaimsJws(token)
                .getBody();
    }

    public static String getSecretForDownstream() {
        return SECRET;   // user-service 校验时用同一密钥（生产应走配置中心或公钥）
    }
}
```

> **知识点**：JWT 由 Header.Payload.Signature 组成；Gateway 只验证签名与过期，**不在网关查库**；用户状态变更需配合短过期或黑名单。

### 2.3 登录接口（Gateway 上提供）

Gateway 是 WebFlux，用 `RouterFunction` 或 `@RestController` 均可。示例用 Controller：

```java
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/auth")
public class AuthController {

    @PostMapping("/login")
    public Mono<Map<String, Object>> login(@RequestBody Map<String, String> body) {
        String username = body.get("username");
        String password = body.get("password");

        // 演示：固定账号 admin / 123456
        if (!"admin".equals(username) || !"123456".equals(password)) {
            return Mono.error(new RuntimeException("用户名或密码错误"));
        }

        String token = JwtUtil.generateToken("1", username);
        Map<String, Object> result = new HashMap<>();
        result.put("token", token);
        result.put("tokenType", "Bearer");
        return Mono.just(result);
    }
}
```

### 2.4 全局鉴权 GlobalFilter

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.List;

@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    private static final List<String> WHITE_LIST = List.of(
            "/api/auth/",
            "/actuator"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        // 外部路径：/api/auth/login、/api/users/... 
        if (WHITE_LIST.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        String token = authHeader.substring(7);
        try {
            JwtUtil.parseToken(token);
        } catch (Exception e) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // Token 合法，原样转发（Authorization 默认会带到下游）
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100;   // 越早执行越好
    }
}
```

### 2.5 路由：增加 auth 路由

在 `application.yml` 的 `gateway.routes` **最前面**加（RewritePath 把外部 `/api/auth/login` 转到本机 Controller `/auth/login`）：

```yaml
- id: auth-route
  uri: forward:/
  predicates:
    - Path=/api/auth/**
  filters:
    - RewritePath=/api/auth/(?<remaining>.*), /auth/${remaining}

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

> `forward:/` 表示仍由 **Gateway 本进程**处理，不转发到 Nacos 上的其它服务；配合 RewritePath 命中 `@RequestMapping("/auth")` 的登录接口。

---

## 3、order-service：Feign 透传 Token

### 3.1 RequestInterceptor

```java
import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;

@Component
public class FeignTokenInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        ServletRequestAttributes attrs =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attrs == null) {
            return;
        }
        HttpServletRequest request = attrs.getRequest();
        String authorization = request.getHeader("Authorization");
        if (authorization != null) {
            template.header("Authorization", authorization);
        }
    }
}
```

> **知识点**：Feign 在新线程里发 HTTP，`RequestContextHolder` 依赖 Tomcat 把当前请求绑到线程；异步 `@Async` 调用 Feign 时需手动传 Token 或改用 `TransmittableThreadLocal`。

### 3.2 无需改 Feign 接口

`UserClient` 保持不变；拦截器对所有 Feign 请求生效。

---

## 4、user-service：校验 JWT

### 4.1 pom.xml 同样引入 jjwt（三件套）

### 4.2 拦截器

```java
import io.jsonwebtoken.Claims;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class JwtAuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        String auth = request.getHeader("Authorization");
        if (auth == null || !auth.startsWith("Bearer ")) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            return false;
        }
        try {
            Claims claims = JwtUtil.parseToken(auth.substring(7));
            request.setAttribute("userId", claims.getSubject());
            request.setAttribute("username", claims.get("username"));
            return true;
        } catch (Exception e) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            return false;
        }
    }
}
```

`JwtUtil` 与 Gateway **同一套密钥**（演示复制一份到 user-service；生产放 Nacos `common.yaml`）。

### 4.3 注册拦截器

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired
    private JwtAuthInterceptor jwtAuthInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(jwtAuthInterceptor).addPathPatterns("/**");
    }
}
```

### 4.4 Controller 返回当前用户

```java
@GetMapping("/{id}")
public Map<String, Object> getById(@PathVariable Long id, HttpServletRequest request) {
    Map<String, Object> user = new HashMap<>();
    user.put("id", id);
    user.put("name", "张三");
    user.put("requestedBy", request.getAttribute("username"));  // 验证透传成功
    return user;
}
```

---

## 5、完整验证步骤

### 5.1 启动

Nacos → user → order → gateway。

### 5.2 无 Token 应 401

```bash
curl -i http://127.0.0.1:9000/api/users/1
# HTTP/1.1 401 Unauthorized
```

### 5.3 登录拿 Token

```bash
curl -s -X POST http://127.0.0.1:9000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"123456"}'
```

响应示例：

```json
{ "token": "eyJhbG...", "tokenType": "Bearer" }
```

复制 `token` 变量：

```bash
TOKEN="eyJhbG..."
```

### 5.4 带 Token 访问 user

```bash
curl -s http://127.0.0.1:9000/api/users/1 \
  -H "Authorization: Bearer $TOKEN"
```

### 5.5 带 Token 访问 order（内部 Feign 调 user，必须透传）

```bash
curl -s http://127.0.0.1:9000/api/orders/100 \
  -H "Authorization: Bearer $TOKEN"
```

期望 `user.requestedBy` 为 `admin`，说明 Feign 已把 Token 传到 user-service。

---

## 6、生产注意

| 点 | 建议 |
| -- | ---- |
| 密钥 | 放 Nacos，Gateway 与 user 共用；或 RS256 公钥私钥 |
| 白名单 | 登录、验证码、健康检查、静态资源 |
| 用户态 | JWT 无状态；踢人需 Redis 黑名单或极短过期 |
| 内部调用 | 可选 **内部签名 Header**（如 `X-Internal-Token`）防绕过 Gateway 直连 |
| TraceId | 另加 `RequestInterceptor` 透传 `X-Trace-Id`，见 Boot 实践 logback 篇 |

---

## 7、常见问题

| 现象 | 处理 |
| ---- | ---- |
| order 调 user 401 | Feign 未注册 `RequestInterceptor`；或异步线程丢失 RequestContext |
| login 404 | 路由未配 `forward:/auth`；Path 与 Controller `@RequestMapping` 不一致 |
| Gateway 401 但 user 正常 | 白名单路径写错；StripPrefix 导致 Filter 看到的 path 与预期不同 |
| 密钥不一致 | Gateway 签发与 user 校验必须用同一 secret |

---

**下一篇**：[4、Sentinel 限流熔断与 Nacos 持久化](./4、Sentinel限流熔断与Nacos持久化.md)
