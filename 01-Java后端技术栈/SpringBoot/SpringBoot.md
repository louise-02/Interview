# 统一返回值

```java
import lombok.Data;

import java.io.Serializable;

@Data
public class Result<T> implements Serializable {
    private Integer code;
    private String msg;
    private T data;

    private Result(Integer code, String msg, T data) {
        this.code = code;
        this.msg = msg;
        this.data = data;
    }

    public static <T> Result<T> success(T data) {
        return new Result<>(200, "success", data);
    }

    public static <T> Result<T> success() {
        return success(null);
    }

    public static <T> Result<T> error(Integer code, String msg) {
        return new Result<>(code, msg, null);
    }

    public static <T> Result<T> error(String msg) {
        return error(500, msg);
    }
}
```

# 异常统一处理

自定义异常

```java
import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {
    private final Integer code;
    private final String msg;

    public BusinessException(Integer code, String msg) {
        super(msg); // 把 msg 传给父类，打印堆栈时能看到具体错误
        this.code = code;
        this.msg = msg;
    }
}
```

@RestControllerAdvice + @ExceptionHandler

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.dao.DataAccessException;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.validation.ObjectError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.stream.Collectors;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {
    /**
     * 拦截所有其他异常
     */
    @ExceptionHandler(value = Exception.class)
    public Result<?> exception(Exception e) {
        log.error("Exception ：", e);
        return Result.error("服务器异常，请联系管理员");
    }

    /**
     * 拦截自定义异常
     */
    @ExceptionHandler(value = BusinessException.class)
    public Result<?> businessException(BusinessException e) {
        log.error("BusinessException ：", e);
        return Result.error(e.getCode(), e.getMsg());
    }

    /**
     * 拦截参数校验错误
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<?> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getAllErrors().stream().map(ObjectError::getDefaultMessage)
                .collect(Collectors.joining(", "));
        log.error("MethodArgumentNotValidException ：{}", message);
        return Result.error(400, message);
    }

    /**
     * 拦截数据库唯一约束冲突（Unique Index）
     */
    @ExceptionHandler(DuplicateKeyException.class)
    public Result<?> handleDuplicateKeyException(DuplicateKeyException e) {
        log.error("唯一索引冲突: ", e);
        return Result.error(400, "该数据已存在，请勿重复操作");
    }

    /**
     * 拦截所有数据库相关报错
     */
    @ExceptionHandler(DataAccessException.class)
    public Result<?> handleDataAccessException(DataAccessException e) {
        log.error("数据库操作异常: ", e);
        return Result.error(500, "数据库服务繁忙");
    }
}

```



# 拦截器 HandlerInterceptor

拦截器

```java
@Component
public class MyInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 核心：请求到达 Controller 之前执行
        // 用途：鉴权、限流、从 Header 解析用户信息并存入 ThreadLocal
        // 返回：true 放行；false 拦截（请求在此终结）
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) {
        // 核心：Controller 执行完，但视图还没渲染时执行
        // 用途：对返回的 ModelAndView 进行统一处理（JSON 时代用得较少）
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        // 核心：整个请求彻底结束（视图渲染完/响应已发出）后执行
        // 用途：资源清理（ThreadLocal.remove()）、性能监控（计算耗时）、异常记录
    }
}
```

注册配置

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new MyInterceptor())
                .addPathPatterns("/**")        // 指定拦截范围
                .excludePathPatterns("/login"); // 指定放行路径
    }
}
```



# 参数解析器 ArgumentResolver

参数解析器

```java
@Component
public class MyParamResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        // 核心：判断参数是否需要由我处理
        // 常用：判断参数上是否有某个注解，或参数是否为某个特定类型
        return parameter.hasParameterAnnotation(MyAnnotation.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        // 核心：具体的转换逻辑
        // 用途：解析 Token、解密参数、从数据库查出对象、聚合分页参数等
        // 返回：返回的对象将直接赋值给 Controller 的形参
        return "加工后的参数值";
    }
}
```

注册配置

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new MyParamResolver());
    }
}
```

# 返回值解析器 ReturnValueHandler

返回值解析器

```java
public class MyResultHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        // 核心：判断返回类型是否由我处理
        // 常用：判断返回对象类型，或类/方法上是否有特定注解
        return !returnType.getParameterType().equals(Result.class);
    }

    @Override
    public void handleReturnValue(Object returnValue, MethodParameter returnType,
                                  ModelAndViewContainer mavContainer, NativeWebRequest webRequest) {
        // 核心：处理返回值逻辑
        // 用途：统一包装 Result 对象、数据加密后再输出、转换特定的响应格式
        // 注意：如果要直接写出 JSON，需操作 HttpServletResponse 并标记请求已处理
        mavContainer.setRequestHandled(true); 
    }
}
```

注册配置

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers) {
        handlers.add(new MyResultHandler());
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

