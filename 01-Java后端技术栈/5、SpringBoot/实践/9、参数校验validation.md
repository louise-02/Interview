见 [4、Web与数据层整合](../4、Web与数据层整合.md#5整合-validation)。异常处理与 [1、统一返回值与异常](./1、统一返回值与异常.md) 配合。

---

# 1、依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Boot 2.3+ **不再**随 `spring-boot-starter-web` 传递 validation，需**显式引入**。

---

# 2、请求 DTO + 注解

```java
import lombok.Data;

import javax.validation.constraints.*;
import java.math.BigDecimal;

@Data
public class OrderCreateRequest {

    @NotNull(message = "用户 ID 不能为空")
    private Long userId;

    @NotBlank(message = "商品名称不能为空")
    @Size(max = 100, message = "商品名称最多 100 字")
    private String productName;

    @NotNull(message = "金额不能为空")
    @DecimalMin(value = "0.01", message = "金额至少 0.01")
    private BigDecimal amount;

    @Min(value = 1, message = "数量至少为 1")
    @Max(value = 9999, message = "数量不能超过 9999")
    private Integer quantity;

    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String mobile;
}
```

**常用注解**

| 注解 | 作用 |
| ---- | ---- |
| `@NotNull` | 不能 null（对象、包装类） |
| `@NotBlank` | 字符串非 null 且 trim 后非空 |
| `@NotEmpty` | 集合/数组/字符串非空 |
| `@Size` | 长度范围 |
| `@Min` / `@Max` | 数值范围 |
| `@Email` | 邮箱格式 |
| `@Past` / `@Future` | 日期过去/将来 |

---

# 3、Controller 触发校验

## 3.1 RequestBody

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public Result<Void> create(@Valid @RequestBody OrderCreateRequest request) {
        // 能进来说明校验已通过
        return Result.success();
    }
}
```

**必须**同时有 `@Valid`（或 `@Validated`）和 DTO 上的约束注解。

## 3.2 路径 / 查询参数

类上 `@Validated`，方法参数上加约束：

```java
@RestController
@Validated
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public Result<?> get(@PathVariable @Min(1) Long id) {
        return Result.success();
    }

    @GetMapping("/search")
    public Result<?> search(
            @RequestParam @NotBlank String keyword,
            @RequestParam @Min(1) @Max(100) Integer pageSize) {
        return Result.success();
    }
}
```

路径/查询参数失败抛 **`ConstraintViolationException`**，与 Body 的 **`MethodArgumentNotValidException`** 不同，需分别处理。

---

# 4、全局异常（补全 Handler）

在 [统一返回值与异常](./1、统一返回值与异常.md) 的 `GlobalExceptionHandler` 中增加：

```java
import javax.validation.ConstraintViolation;
import javax.validation.ConstraintViolationException;

@ExceptionHandler(ConstraintViolationException.class)
public Result<?> handleConstraintViolation(ConstraintViolationException e) {
    String message = e.getConstraintViolations().stream()
            .map(ConstraintViolation::getMessage)
            .collect(Collectors.joining(", "));
    return Result.error(400, message);
}
```

`MethodArgumentNotValidException` 处理器已在实践 1 中。

---

# 5、分组校验

同一 DTO 不同场景用不同规则（创建 vs 更新）：

```java
public interface CreateGroup {}
public interface UpdateGroup {}

@Data
public class UserSaveRequest {

    @Null(groups = CreateGroup.class, message = "创建时不能传 id")
    @NotNull(groups = UpdateGroup.class, message = "更新时 id 必填")
    private Long id;

    @NotBlank(groups = {CreateGroup.class, UpdateGroup.class})
    private String username;
}
```

```java
@PostMapping
public Result<?> create(@Validated(CreateGroup.class) @RequestBody UserSaveRequest req) {
    return Result.success();
}

@PutMapping
public Result<?> update(@Validated(UpdateGroup.class) @RequestBody UserSaveRequest req) {
    return Result.success();
}
```

---

# 6、嵌套校验

```java
@Data
public class OrderBatchRequest {

    @NotEmpty(message = "订单列表不能为空")
    @Valid   // 必须加 @Valid，才会校验列表元素内部字段
    private List<OrderCreateRequest> orders;
}
```

---

# 7、自定义校验注解

## 注解

```java
import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = IdCardValidator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface IdCard {

    String message() default "身份证号不合法";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

## 校验器

```java
import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

public class IdCardValidator implements ConstraintValidator<IdCard, String> {

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null || value.isBlank()) {
            return true;   // 是否必填交给 @NotBlank
        }
        // 简化示例：18 位
        return value.matches("^\\d{17}[\\dXx]$");
    }
}
```

## 使用

```java
@IdCard(message = "身份证号格式错误")
private String idCard;
```

---

# 8、手动触发校验（Service 层）

```java
import org.springframework.stereotype.Service;

import javax.validation.Validator;
import java.util.stream.Collectors;

@Service
public class OrderService {

    @Autowired
    private Validator validator;

    public void validate(OrderCreateRequest request) {
        var violations = validator.validate(request);
        if (!violations.isEmpty()) {
            String msg = violations.stream()
                    .map(v -> v.getMessage())
                    .collect(Collectors.joining(", "));
            throw new BusinessException(400, msg);
        }
    }
}
```

适合 Excel 导入、MQ 消息等非 HTTP 入参。

---

# 9、验证

```bash
# 缺字段 → 400
curl -s -X POST http://127.0.0.1:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"amount":0}'

# 期望：{"code":400,"msg":"商品名称不能为空, 金额至少 0.01, ..."}
```

---

# 10、注意

| 点 | 说明 |
| -- | ---- |
| `@Valid` vs `@Validated` | 后者支持**分组**；前者 JSR-303 标准 |
| 基本类型 | `int` 无法 `@NotNull`，用 `Integer` |
| 国际化 | `message = "{order.amount.min}"` + `ValidationMessages.properties` |
| 性能 | 常规接口开销可忽略；批量导入用手动 `Validator` 批量校验 |
