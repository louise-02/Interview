# 1、核心概念与 Filter 链

> 生产要建哪些类、各做什么，见 **[0、生产集成完整方案](./0、生产集成完整方案.md)**。本章讲原理。

## 1、三大职责

| 职责 | 英文 | 做什么 |
| ---- | ---- | ------ |
| 认证 | Authentication | 确认身份（登录成功 → 得到 `Authentication`） |
| 授权 | Authorization | 判断是否有权访问资源 |
| 攻击防护 | Protection | CSRF、安全响应头、Session 固定等 |

---

## 2、SecurityContext 与 ThreadLocal

认证成功后，用户信息放在 **当前线程** 的 `SecurityContext` 里：

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();                    // 用户名
Object principal = auth.getPrincipal();              // 往往是 UserDetails 或 JWT 解析结果
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
```

- **同一次请求**内 Controller、Service、`@PreAuthorize` 都读同一份 Context。
- 请求结束 Filter 会清理，避免线程池复用脏数据。
- **不要在子线程直接继承**父线程 Context；异步需 `DelegatingSecurityContextRunnable` 或手动传递。

---

## 3、Authentication 里有什么

| 字段 | 说明 |
| ---- | ---- |
| `principal` | 用户身份对象（`UserDetails`、JWT Claims、String 用户名等） |
| `credentials` | 凭证（密码、Token）；认证成功后常被清空 |
| `authorities` | 权限集合 |
| `authenticated` | 是否已认证 |

未登录时通常是 `AnonymousAuthenticationToken`（匿名用户），是否允许访问由授权规则决定。

---

## 4、Filter 链（为什么比 Interceptor 更早）

Spring Security 本质是往 Servlet 容器注册 **一组 Filter**，在 `DispatcherServlet` **之前**执行。

常见 Filter（顺序因配置而异，理解即可）：

| Filter（示例） | 作用 |
| -------------- | ---- |
| `SecurityContextPersistenceFilter` | 请求进来恢复/创建 SecurityContext |
| `LogoutFilter` | 处理登出 |
| `UsernamePasswordAuthenticationFilter` | 表单登录 POST `/login` |
| `BearerTokenAuthenticationFilter` | OAuth2 Resource Server 解析 JWT |
| `BasicAuthenticationFilter` | HTTP Basic |
| `AuthorizationFilter` | **授权**：当前用户能否访问该 URL（Security 6） |

```
客户端
  → [Security Filters...]  认证 + 授权（URL 级）
  → DispatcherServlet
  → [HandlerInterceptor]     业务拦截（可选）
  → Controller
  → [@PreAuthorize AOP]      方法级授权（可选）
```

**结论**：登录鉴权应放在 Security，不要用 Interceptor 重复造轮子；Interceptor 适合操作日志、租户 ID 解析等。

---

## 5、UserDetails 与 UserDetailsService

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

典型流程（表单或 JSON 登录）：

1. 客户端提交用户名、密码  
2. `DaoAuthenticationProvider` 调 `UserDetailsService.loadUserByUsername`  
3. `PasswordEncoder.matches` 校验密码  
4. 成功 → 构造 `Authentication` 写入 `SecurityContextHolder`  

业务用户表与 Security 的桥梁：**实现 UserDetailsService 查库**，必要时自定义 `UserDetails` 携带 userId、部门等。

---

## 6、PasswordEncoder

**禁止明文存密码。** 常用 BCrypt：

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

// 注册时
String hash = passwordEncoder.encode(rawPassword);

// 登录校验由 Security 内部 matches(raw, hash) 完成
```

---

## 7、两种授权粒度

| 粒度 | 配置位置 | 示例 |
| ---- | -------- | ---- |
| URL / HTTP | `SecurityFilterChain` 里 `authorizeHttpRequests` | `/admin/**` 要 `ROLE_ADMIN` |
| 方法 | `@EnableMethodSecurity` + `@PreAuthorize` | `@PreAuthorize("hasAuthority('order:approve')")` |

推荐：**URL 做粗粒度**（静态资源、Actuator、登录接口白名单），**方法做细粒度**（业务权限）。

---

## 8、调试技巧

开启 Security 调试日志：

```yaml
logging:
  level:
    org.springframework.security: DEBUG
```

看 Filter 链与认证失败原因（401 未认证 vs 403 已认证无权限）。

Boot 可临时暴露（仅开发）：

```yaml
management:
  endpoint:
    mappings:
      enabled: true
```

配合日志理解哪个 Filter 拦截了请求。
