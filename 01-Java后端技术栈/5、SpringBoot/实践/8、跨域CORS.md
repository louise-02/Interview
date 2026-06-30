浏览器**同源策略**会拦截前端（如 `http://localhost:5173`）访问后端 API（如 `http://localhost:8080`）的响应。后端需返回 CORS 相关 Header 才能放行。

---

# 1、三种做法怎么选

| 方式 | 适用 |
| ---- | ---- |
| **WebMvcConfigurer#addCorsMappings** | 全局、最常用 |
| **@CrossOrigin** | 单个 Controller / 方法 |
| **CorsFilter** | 需要比 MVC 更早处理、或非 Spring MVC 栈 |

微服务架构下，也可在 **Gateway** 统一配 CORS，业务服务不再配（见文末）。

---

# 2、全局配置（推荐）

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")           // 哪些路径允许跨域
                .allowedOriginPatterns("*")      // 允许的来源；生产写具体域名
                // .allowedOrigins("http://localhost:5173", "https://www.example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .exposedHeaders("Authorization", "Content-Disposition")  // 前端可读到的响应头
                .allowCredentials(true)          // 允许带 Cookie；此时不能用 allowedOrigins("*")
                .maxAge(3600);                   // 预检 OPTIONS 缓存秒数
    }
}
```

> **知识点 — 简单请求 vs 预检请求**
>
> - 简单 GET/POST（特定 Content-Type）直接发。
> - 带自定义 Header、`PUT/DELETE`、JSON 等会先发 **OPTIONS 预检**，通过后再发真实请求。
> - `maxAge` 减轻预检次数。

---

# 3、按环境区分（生产收紧）

```yaml
cors:
  allowed-origins:
    - http://localhost:5173
    - https://admin.example.com
```

```java
@Configuration
@ConfigurationProperties(prefix = "cors")
public class CorsProperties {
    private List<String> allowedOrigins = new ArrayList<>();
    // getter / setter
}

@Configuration
@RequiredArgsConstructor
public class CorsConfig implements WebMvcConfigurer {

    private final CorsProperties corsProperties;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins(corsProperties.getAllowedOrigins().toArray(new String[0]))
                .allowedMethods("*")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

---

# 4、Controller 级 @CrossOrigin

```java
@CrossOrigin(origins = "http://localhost:5173", maxAge = 3600)
@RestController
@RequestMapping("/api/demo")
public class DemoController {

    @CrossOrigin(origins = "https://partner.com")   // 方法级覆盖
    @GetMapping("/partner")
    public String partner() {
        return "ok";
    }
}
```

适合**个别开放接口**（如对外 webhook）；全局接口仍建议 `WebMvcConfigurer` 统一管理。

---

# 5、CorsFilter（Filter 层）

拦截器 / Spring Security 之前就要 CORS 时：

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import org.springframework.web.filter.CorsFilter;

@Configuration
public class GlobalCorsFilter {

    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedOriginPattern("*");
        config.addAllowedHeader("*");
        config.addAllowedMethod("*");
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }
}
```

---

# 6、与 Spring Security 配合

若用了 Security，仅配 MVC 可能不够，需：

```java
http.cors();   // 使用 CorsConfigurationSource Bean
```

或：

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.cors(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable());
    return http.build();
}
```

---

# 7、Gateway 统一跨域（微服务）

业务服务不配 CORS，在 Gateway `application.yml`：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOriginPatterns: "*"
            allowedMethods: "*"
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
```

避免每个服务重复配、重复返回 CORS 头。

---

# 8、验证

浏览器 F12 → Network，看响应头是否含：

```text
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Allow-Credentials: true
```

或用 curl 模拟预检：

```bash
curl -i -X OPTIONS http://127.0.0.1:8080/api/users/1 \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET"
```

应返回 `204` 或 `200` 且带 `Access-Control-Allow-*`。

---

# 9、常见问题

| 现象 | 原因 |
| ---- | ---- |
| 仍报 CORS | 实际 401/500，浏览器也报成 CORS；先看 Network 真实状态码 |
| credentials + `*` | `allowCredentials(true)` 时 `allowedOrigins` 不能为 `*`，用 `allowedOriginPatterns` 或枚举域名 |
| 重复 Header | Gateway 和业务都配了 CORS，出现两个 `Allow-Origin` |
| 拦截器拦 OPTIONS | 拦截器 `excludePathPatterns` 或 Filter 层处理 CORS |
