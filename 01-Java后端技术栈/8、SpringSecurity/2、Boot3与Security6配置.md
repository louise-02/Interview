# 2、Boot3 与 Security6 配置

先按 **[0、生产集成完整方案](./0、生产集成完整方案.md)** 把类与 Filter 链配齐；本章展开每一项配置的含义与变体。

---

## 1、最小可用配置

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 关闭 CSRF（纯 REST + JWT 时常关；有 Cookie Session 时要谨慎）
            .csrf(csrf -> csrf.disable())
            // 授权规则
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            // 无自定义登录接口时，会启用默认 formLogin（前后端分离一般关掉，见第 3 章 JWT）
            .formLogin(form -> form.disable())
            .httpBasic(basic -> basic.disable());
        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 2、requestMatchers 常用写法

| 写法 | 说明 |
| ---- | ---- |
| `requestMatchers("/api/public/**").permitAll()` | 白名单 |
| `requestMatchers(HttpMethod.GET, "/api/users/**").hasAuthority("user:read")` | 按 HTTP 方法 |
| `requestMatchers("/api/admin/**").hasRole("ADMIN")` | 角色（自动加 `ROLE_` 前缀） |
| `anyRequest().authenticated()` | 其余都要登录 |

Security 6 用 **`requestMatchers`** 替代 5.x 的 `antMatchers` / `mvcMatchers`。

---

## 3、Session 与无状态

**前后端分离 + JWT**（常见）：

```java
.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
```

不创建 HttpSession，认证信息只靠 Token。

**传统 Session 登录**：

```java
.sessionManagement(session -> session
    .maximumSessions(1)                    // 同一用户最大会话数（可选）
    .maxSessionsPreventsLogin(false))     // false=踢掉旧会话
```

---

## 4、CSRF

- **Cookie + Session + 浏览器表单**：应开启 CSRF，或前后端约定 CSRF Token。  
- **JWT 放 Header、无 Cookie 认证**：通常 **disable CSRF**（本项目 REST 常见做法）。

```java
.csrf(csrf -> csrf.disable())
```

---

## 5、CORS 与 Security

仅配 `WebMvcConfigurer` 不够，Security 链也要开 CORS：

```java
http.cors(Customizer.withDefaults());
```

并提供 Bean（与 [SpringBoot/实践/8、跨域CORS](../../SpringBoot/实践/8、跨域CORS.md) 一致）：

```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOriginPatterns(List.of("*"));
    config.setAllowedMethods(List.of("*"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

微服务场景更推荐 **Gateway 统一 CORS**，业务服务可简化。

---

## 6、异常处理：401 / 403 JSON

默认会 302 到登录页或 Whitelabel，前后端分离需自定义：

```java
.exceptionHandling(ex -> ex
    .authenticationEntryPoint((req, resp, e) -> {
        resp.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        resp.setContentType(MediaType.APPLICATION_JSON_VALUE);
        resp.getWriter().write("{\"code\":401,\"msg\":\"未登录或 Token 无效\"}");
    })
    .accessDeniedHandler((req, resp, e) -> {
        resp.setStatus(HttpServletResponse.SC_FORBIDDEN);
        resp.setContentType(MediaType.APPLICATION_JSON_VALUE);
        resp.getWriter().write("{\"code\":403,\"msg\":\"无权限\"}");
    }));
```

| 状态码 | 含义 |
| ------ | ---- |
| **401** | 未认证（没 Token、Token 过期） |
| **403** | 已认证但权限不足 |

---

## 7、多 SecurityFilterChain（高级）

不同 URL 用不同安全策略（例如 `/api/**` JWT，`/swagger/**` 另一套）：

```java
@Bean
@Order(1)
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        // ... JWT 配置
        ;
    return http.build();
}

@Bean
@Order(2)
SecurityFilterChain defaultChain(HttpSecurity http) throws Exception {
    // 其他路径
    return http.build();
}
```

---

## 8、Boot 2 迁移对照

| Security 5（Boot 2） | Security 6（Boot 3） |
| -------------------- | -------------------- |
| `extends WebSecurityConfigurerAdapter` | `@Bean SecurityFilterChain` |
| `authorizeRequests()` | `authorizeHttpRequests()` |
| `antMatchers("/x/**")` | `requestMatchers("/x/**")` |
| `csrf().disable()` | `csrf(csrf -> csrf.disable())` |
| `javax.servlet.*` | `jakarta.servlet.*` |

旧项目可先在 5.7 改为 `SecurityFilterChain` Bean，再升 Boot 3。

---

## 9、配置属性（补充）

```yaml
spring:
  security:
    user:
      name: admin          # 仅开发：默认用户（生产不要用）
      password: admin
      roles: ADMIN
```

生产环境应用 **数据库用户 + UserDetailsService**，不要依赖默认用户。
