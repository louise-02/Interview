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



# logback

xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 引入spring boot默认的logback配置文件 获取输出格式 -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- CONSOLE -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 采用Spring boot中默认的控制台彩色日志输出模板 -->
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>${CONSOLE_LOG_CHARSET}</charset>
        </encoder>
    </appender>
  
    <!-- 定义变量 -->
    <property name="LOG_HOME" value="./logs" />

    <!-- 可以用自定义的 SchedulerLogAppender -->
    <!-- TOTAL 接收全部信息 取决于日志级别 -->
    <appender name="TOTAL_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${LOG_HOME}/total.log</file>

        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- 归档的日志文件的路径 -->
            <fileNamePattern>${LOG_HOME}/total/total-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <!-- 保留最近 7 天的日志 -->
            <maxHistory>7</maxHistory>
            <!-- 单个文件最大 200MB -->
            <maxFileSize>200MB</maxFileSize>
            <!-- 超过 5GB 时删除旧日志 -->
            <totalSizeCap>5GB</totalSizeCap>
        </rollingPolicy>

        <!-- 追加方式记录日志 -->
        <append>true</append>

        <!-- 日志文件的格式 -->
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>${FILE_LOG_CHARSET}</charset>
        </encoder>
    </appender>

    <!-- ERROR 只接收 ERROR -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${LOG_HOME}/error.log</file>

        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- 归档的日志文件的路径 -->
            <fileNamePattern>${LOG_HOME}/error/error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <!-- 保留最近 7 天的日志 -->
            <maxHistory>7</maxHistory>
            <!-- 单个文件最大 200MB -->
            <maxFileSize>200MB</maxFileSize>
            <!-- 超过 5GB 时删除旧日志 -->
            <totalSizeCap>5GB</totalSizeCap>
        </rollingPolicy>

        <!-- 日志文件的格式 -->
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>${FILE_LOG_CHARSET}</charset>
        </encoder>

        <!-- 追加方式记录日志 -->
        <append>true</append>

        <!-- 此日志文件ERROR及以上级别的 -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
    </appender>

    <logger name="com.louise" level="DEBUG"/>

    <!-- 默认配置（非local环境） -->
    <springProfile name="!local">
        <root level="INFO">
            <appender-ref ref="TOTAL_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
            <!-- 生产环境不输出到控制台 -->
        </root>
    </springProfile>

    <!-- local环境特殊配置 -->
    <springProfile name="local">
        <root level="DEBUG">  <!-- local环境可以更详细 -->
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="TOTAL_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
        </root>
    </springProfile>

</configuration>
```

## 日志追踪

修改 xml 增加 traceId

```xml
    <!--日志格式应用spring boot默认的格式，也可以自己更改-->
<!--    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>-->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
    <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
    <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />

    <property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr(%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) [traceId:%X{traceId:-N/A}] %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
    <property name="FILE_LOG_PATTERN" value="${FILE_LOG_PATTERN:-%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}} ${LOG_LEVEL_PATTERN:-%5p} [traceId:%X{traceId:-N/A}] ${PID:- } --- [%t] %-40.40logger{39} : %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
```

增加拦截器

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.UUID;

@Component
public class SlowRequestInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(SlowRequestInterceptor.class);

    // 超时时间阈值（毫秒）
    private static final long THRESHOLD_MS = 10_000;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 生成唯一 traceId
        String traceId = UUID.randomUUID().toString();
        MDC.put("traceId", traceId);
        // 记录请求开始时间
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        String traceId = MDC.get("traceId");
        // 请求完成后清理 MDC
        MDC.remove("traceId");

        Long startTime = (Long) request.getAttribute("startTime");
        if (startTime != null) {
            long duration = System.currentTimeMillis() - startTime;
            if (duration > THRESHOLD_MS) {
                String uri = request.getRequestURI();
                int status = response.getStatus();
                log.info("慢请求警告: [{} {}] 耗时 {} ms, 响应状态: {}", traceId, uri, duration, status);
            }
        }
    }
}
```

增加 WebConfig

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {

    @Autowired
    private SlowRequestInterceptor slowRequestInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(slowRequestInterceptor)
                .addPathPatterns("/**"); // 拦截所有请求
    }
}
```

