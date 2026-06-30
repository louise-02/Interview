# 4、JWT 与 Resource Server

> 生产默认走 **jjwt + JwtAuthenticationFilter**，完整代码见 **[0、生产集成完整方案](./0、生产集成完整方案.md)** §6。

前后端分离最常见：**登录接口发 JWT**，后续请求 `Authorization: Bearer <token>`，服务端**无 Session**。

两种实现路线：

| 路线 | 适用 |
| ---- | ---- |
| **OAuth2 Resource Server**（官方） | 标准 JWT、`spring-boot-starter-oauth2-resource-server` |
| **自定义 JwtAuthenticationFilter** | 完全自控签发/校验逻辑（与现有 Gateway 实践一致） |

---

## 1、OAuth2 Resource Server（推荐新项目评估）

### 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### 配置（对称密钥或 JWK）

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com    # 或 jwk-set-uri
          # 本地 HS256 示例（开发）：
          # 需配合 @Bean JwtDecoder 自定义
```

### SecurityFilterChain

```java
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/auth/login").permitAll()
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
    return http.build();
}
```

`BearerTokenAuthenticationFilter` 自动解析 JWT，成功后 `Authentication` 的 principal 多为 `Jwt` 对象。

取用户名：

```java
@GetMapping("/me")
public String me(@AuthenticationPrincipal Jwt jwt) {
    return jwt.getSubject();
}
```

---

## 2、自定义 JWT Filter（与 Gateway 文档一致）

签发在登录接口（见 [3、认证体系](./3、认证体系.md)），校验用 Filter：

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenService jwtTokenService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            try {
                String username = jwtTokenService.parseUsername(token);
                if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                    UserDetails user = userDetailsService.loadUserByUsername(username);
                    UsernamePasswordAuthenticationToken auth =
                        new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            } catch (Exception e) {
                // Token 无效：不设置 Authentication，后续返回 401
            }
        }
        chain.doFilter(request, response);
    }
}
```

注册到 Filter 链：

```java
http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
```

JWT 工具（jjwt 示例思路，与 [SpringCloud/实践/3](../../SpringCloud/实践/3、Gateway鉴权与Feign透传Token.md) 可共用同一套密钥规范）：

- 登录成功：`createToken(username, authorities, expire)`
- 请求进入：`parse` 验签 + 过期时间
- 密钥长度满足 HS256 要求；生产放配置中心，不要硬编码

---

## 3、JWT 内放什么

| 建议放 | 不建议放 |
| ------ | -------- |
| `sub`（用户名或 userId） | 密码 |
| 过期时间 `exp` | 大量业务字段（Token 变大） |
| 必要 roles（少而精） | 可改敏感数据（JWT 可解码，非加密） |

权限变更后旧 Token 仍有效直到过期 → 重要变更可配合 **Redis 黑名单** 或 **短过期 + Refresh Token**。

---

## 4、Refresh Token（可选）

| Token | 作用 |
| ----- | ---- |
| Access Token | 短过期（15min～2h），访问 API |
| Refresh Token | 长过期，仅换 Access Token；存 HttpOnly Cookie 或安全存储 |

实现：单独 `/api/auth/refresh` 校验 Refresh Token 后发新 Access Token。

---

## 5、与 Gateway 分工

| 层 | 职责 |
| -- | ---- |
| **Gateway** | 统一入口鉴权、限流、CORS；可校验 JWT 或只转发 |
| **业务服务** | 再次校验 JWT（零信任）或信任网关传递的身份 Header（需网关 strip 外部伪造头） |

详见 [Gateway 鉴权实践](../../SpringCloud/实践/3、Gateway鉴权与Feign透传Token.md)。

---

## 6、Feign 透传 Token

Feign 默认不带 `Authorization`，需 `RequestInterceptor` 从当前请求取 Header 传递（Gateway 文档已有完整示例）。

---

## 7、常见坑

| 现象 | 排查 |
| ---- | ---- |
| 401 但 Token 看起来对 | 时钟偏移；密钥不一致；issuer 不匹配 |
| 403 不是 401 | 已认证，缺 authority → 查 `@PreAuthorize` / URL 规则 |
| 每次请求都查库 | Filter 里 `loadUserByUsername` 可改为只信 JWT claims + 缓存 |
| Token 放 URL | 易泄露，只用 Header |
