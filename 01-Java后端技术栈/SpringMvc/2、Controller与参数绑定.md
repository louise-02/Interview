# 2、Controller 与参数绑定

## 1、参数绑定方式

| 注解 | 来源 |
| ---- | ---- |
| `@RequestParam` | Query，可 defaultValue、required |
| `@PathVariable` | URI 模板 |
| `@RequestBody` | Body（JSON） |
| `@RequestHeader` | Header |
| `@CookieValue` | Cookie |
| `@ModelAttribute` | 表单字段绑定到对象 |
| 无注解 + 简单类型 | 按名称匹配 RequestParam |

## 2、校验 JSR-303

```java
public Result<?> create(@Valid @RequestBody UserCreateDTO dto) { }

public class UserCreateDTO {
    @NotBlank
    private String name;
    @Min(0)
    private Integer age;
}
```

配合 `@RestControllerAdvice` + `MethodArgumentNotValidException` 统一返回 400（见 SpringBoot 实践文档）。

## 3、统一响应

业务层返回 `Result<T>`，由 `@RestController` 序列化为 JSON；异常由全局异常处理器转换。

## 4、文件上传

`multipart/form-data` + `@RequestParam MultipartFile file`；大小限制 `spring.servlet.multipart.max-file-size`。

## 5、内容协商

`Accept` 头或 `produces` / `consumes` 指定 JSON、XML 等。

## 6、常见面试题

**1. @RequestBody 和 @RequestParam 区别？**

前者读 Body（常 JSON）；后者读 Query/Form 单个参数。

**2. 参数绑定失败怎么处理？**

类型不匹配 400；校验失败 `@Valid` + 全局异常处理。

**3. 自定义参数解析？**

实现 `HandlerMethodArgumentResolver` + 注册 `WebMvcConfigurer.addArgumentResolvers`（SpringBoot.md 有示例）。
