# logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>${CONSOLE_LOG_CHARSET}</charset>
        </encoder>
    </appender>

    <property name="LOG_HOME" value="./logs" />

    <appender name="TOTAL_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/total.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_HOME}/total/total-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>200MB</maxFileSize>
            <totalSizeCap>5GB</totalSizeCap>
        </rollingPolicy>
        <append>true</append>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>${FILE_LOG_CHARSET}</charset>
        </encoder>
    </appender>

    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/error.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_HOME}/error/error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>200MB</maxFileSize>
            <totalSizeCap>5GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
            <charset>${FILE_LOG_CHARSET}</charset>
        </encoder>
        <append>true</append>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
    </appender>

    <logger name="com.louise" level="DEBUG"/>

    <springProfile name="!local">
        <root level="INFO">
            <appender-ref ref="TOTAL_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
        </root>
    </springProfile>

    <springProfile name="local">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="TOTAL_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
        </root>
    </springProfile>
</configuration>
```

# 日志格式增加 traceId

在 configuration 内增加：

```xml
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
    <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
    <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />

    <property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr(%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) [traceId:%X{traceId:-N/A}] %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
    <property name="FILE_LOG_PATTERN" value="${FILE_LOG_PATTERN:-%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}} ${LOG_LEVEL_PATTERN:-%5p} [traceId:%X{traceId:-N/A}] ${PID:- } --- [%t] %-40.40logger{39} : %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
```

# TraceId 拦截器

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.UUID;

@Component
public class SlowRequestInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(SlowRequestInterceptor.class);
    private static final long THRESHOLD_MS = 10_000;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        MDC.put("traceId", UUID.randomUUID().toString());
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        String traceId = MDC.get("traceId");
        MDC.remove("traceId");
        Long startTime = (Long) request.getAttribute("startTime");
        if (startTime != null) {
            long duration = System.currentTimeMillis() - startTime;
            if (duration > THRESHOLD_MS) {
                log.info("慢请求警告: [{} {}] 耗时 {} ms, 状态: {}",
                        traceId, request.getRequestURI(), duration, response.getStatus());
            }
        }
    }
}
```

注册（Boot 3 请用 `WebMvcConfigurer` 替代已废弃的 `WebMvcConfigurerAdapter`）：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private SlowRequestInterceptor slowRequestInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(slowRequestInterceptor).addPathPatterns("/**");
    }
}
```

# 注意

- 文件名必须是 **logback-spring.xml** 才能用 `<springProfile>`。
- 网关若已有 TraceId，拦截器里应优先读 Header 再写入 MDC，便于链路串联。
