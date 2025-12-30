# 异常统一处理

@RestControllerAdvice + @ExceptionHandler

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {
    /**
     * 处理所有业务异常
     */
    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<Map<String, Object>> handleBusinessException(RuntimeException e) {
        log.error("业务异常: {}", e.getMessage(), e);

        Map<String, Object> result = new HashMap<>();
        result.put("code", 400);
        result.put("message", e.getMessage());
        result.put("success", false);
        result.put("timestamp", System.currentTimeMillis());

        return ResponseEntity.badRequest().body(result);
    }

    /**
     * 处理所有其他异常（兜底）
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleAllException(Exception e) {
        log.error("系统异常: {}", e.getMessage(), e);

        Map<String, Object> result = new HashMap<>();
        result.put("code", 500);
        result.put("message", "系统繁忙，请稍后重试");
        result.put("success", false);
        result.put("timestamp", System.currentTimeMillis());

        return ResponseEntity.status(500).body(result);
    }
}
```

