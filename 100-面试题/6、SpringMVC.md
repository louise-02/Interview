# SpringMVC 面试题

本文件可**单独阅读**，每题含完整参考答案。

---

### 1. SpringMVC 一次请求完整流程？

请求 → **Filter 链** → **DispatcherServlet** → HandlerMapping 找 Handler → HandlerInterceptor.preHandle → HandlerAdapter 执行 Controller → 返回 ModelAndView 或 @ResponseBody 对象 → HttpMessageConverter 写 JSON → postHandle → 视图渲染（若有）→ afterCompletion → 响应。

---

### 2. 核心组件有哪些？

DispatcherServlet、HandlerMapping、HandlerAdapter、HandlerInterceptor、ViewResolver、HandlerExceptionResolver、HttpMessageConverter。

---

### 3. @RestController 和 @Controller？

@RestController = @Controller + @ResponseBody，方法返回值直接序列化为 JSON/XML 写响应体，不经过视图解析。

---

### 4. @RequestMapping 和 @GetMapping？

GetMapping 是 RequestMapping(method=GET) 的复合注解，更简洁。还有 PostMapping、PutMapping、DeleteMapping 等。

---

### 5. @PathVariable 和 @RequestParam？

@PathVariable 从 URI 模板 `/user/{id}` 取 id。@RequestParam 从 Query String `?name=xx` 或 form 取，可 required、defaultValue。

---

### 6. @RequestBody 和 @RequestParam？

@RequestBody 读 **HTTP Body**（通常 JSON），由 Jackson 反序列化为 Java 对象。@RequestParam 读 URL 参数或表单字段，不能读 JSON Body。

---

### 7. Filter 和 Interceptor 区别与顺序？

Filter 属于 Servlet 规范，在 DispatcherServlet **之前**，可拦截所有请求。Interceptor 属于 Spring MVC，在 DispatcherServlet **内部**，Controller 前后。典型：Filter 做编码/XSS；Interceptor 做登录鉴权、ThreadLocal 清理。

---

### 8. Interceptor 三个方法？

**preHandle**：Controller 前，return false 中断。**postHandle**：Controller 后、视图前（REST 项目少用）。**afterCompletion**：请求结束，适合 **ThreadLocal.remove()**、记录总耗时。

---

### 9. 全局异常怎么处理？

@RestControllerAdvice（或 @ControllerAdvice + @ResponseBody）+ @ExceptionHandler(异常类型)，返回统一 Result JSON。可区分 BusinessException、ValidationException、Exception。

---

### 10. 参数校验怎么做？

DTO 字段加 @NotNull、@NotBlank 等，Controller 参数 @Valid @RequestBody DTO。失败抛 MethodArgumentNotValidException，全局异常处理器返回 400 和错误信息。

---

### 11. 文件上传？

`multipart/form-data`，参数 MultipartFile，配置 `spring.servlet.multipart.max-file-size`。大文件可用分片或直传 OSS。

---

### 12. 跨域 CORS 怎么配？

WebMvcConfigurer.addCorsMappings 设置 allowedOrigins/Methods/Headers。生产网关统一 CORS 时后端可关闭重复配置。注意 allowCredentials 与 `*`  origin 兼容性。

---

### 13. HttpMessageConverter 作用？

在请求/响应体与 Java 对象间转换。MappingJackson2HttpMessageConverter 处理 application/json。可自定义 XML、Protobuf 等。

---

### 14. 404 常见原因？

无匹配的 @RequestMapping；context-path 配错；静态资源未映射；Controller 未被扫描（包路径不对）。

---

### 15. 静态资源怎么配？

WebMvcConfigurer.addResourceHandlers 映射 `/static/**` 到 classpath:/static/。Boot 默认 static、public、resources、META-INF/resources。

---

### 16. 自定义参数解析器？

实现 HandlerMethodArgumentResolver（supportsParameter + resolveArgument），注册到 WebMvcConfigurer.addArgumentResolvers。用于解析 @CurrentUser 等自定义注解。

---

### 17. 转发 forward 和重定向 redirect？

forward 服务器内部跳转，URL 不变，一次请求。redirect 302 新 URL，浏览器再请求，丢 POST body。

---

### 18. RESTful 设计要点？

资源用名词 URI，HTTP 方法表语义：GET 查、POST 增、PUT 全量改、PATCH 部分改、DELETE 删。状态码：200、201、400、401、404、500。

---

### 19. Content-Type 和 Accept？

Content-Type 请求体格式。Accept 期望响应格式。REST 常用 application/json。

---

### 20. 如何避免 POST 重复提交？

Token 表单、幂等键 Idempotency-Key、Redis setnx 短 TTL、前端防抖。服务端业务幂等更根本。

---

### 21. DispatcherServlet 和 Servlet 容器关系？

DispatcherServlet 本身是 Servlet，由 Tomcat 等容器加载，映射 `/` 或 context path，是 Spring MVC 唯一入口（典型配置）。

---

### 22. HandlerMapping 常见实现？

RequestMappingHandlerMapping（@RequestMapping）、BeanNameUrlHandlerMapping（旧）。Boot 主要用注解映射。

---

### 23. 为什么 afterCompletion 要清 ThreadLocal？

Tomcat **线程池复用**线程，不清除下一个请求会读到上一个用户数据，且可能内存泄漏。

---

### 24. @ResponseStatus 作用？

标注在异常类或方法上，指定 HTTP 状态码，如 404、400，配合异常处理器或直接使用。

---

### 25. WebMvcConfigurer 常用重写？

addInterceptors、addCorsMappings、addResourceHandlers、configureMessageConverters、addArgumentResolvers。

---

### 26. Spring MVC 和 WebFlux 区别？

MVC 基于 Servlet，阻塞 IO，线程 per request。WebFlux 反应式非阻塞，RouterFunction/注解，适合高并发 IO，Gateway 基于 WebFlux。

---

### 27. 如何做接口版本？

URL `/v1/user`、Header Accept-Version、参数 version=1。选一种团队统一即可。

---

### 28. JSON 日期格式统一？

Jackson `@JsonFormat` 或全局 ObjectMapper 配置 `JavaTimeModule`、禁用 timestamps、`yyyy-MM-dd HH:mm:ss`。

---

### 29. 大整数 JSON 精度丢失？

JavaScript Number 精度有限，Long 型 ID 前端可能丢精度。序列化为 **String** 或 @JsonSerialize(ToStringSerializer.class)。

---

### 30. Spring MVC 处理流程中何时写 JSON？

Controller 返回对象 → RequestResponseBodyMethodProcessor → MappingJackson2HttpMessageConverter.writeInternal 写入 OutputStream。
