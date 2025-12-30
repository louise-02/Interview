# 1、Springboot 集成

**依赖添加**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

# 4、caffeine 集成

**用途**

1、主要用户在内存中进行缓存，避免高并发请求对 Redis 压力过大

**添加依赖**

```
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>2.9.3</version>
</dependency>
```

**配置文件**

```java
import cn.stylefeng.roses.kernel.config.api.context.ConfigContext;
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.LoadingCache;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.core.StringRedisTemplate;

import java.util.concurrent.TimeUnit;

/**
 * 本地缓存 config
 *
 * @author Louise
 * @since 2025/05/20
 */
@Configuration
public class LocalCacheConfig {
    /**
     * token 是否存在缓存
     *
     * @param stringRedisTemplate stringRedisTemplate
     * @return {@code LoadingCache<String, Boolean> }
     */
    @Bean
    public LoadingCache<String, Boolean> tokenExistCache(@Autowired StringRedisTemplate stringRedisTemplate) {
        return Caffeine.newBuilder()
                .maximumSize(10_000) // 可根据实际请求量调整
                .expireAfterWrite(60, TimeUnit.SECONDS) // 本地缓存过期时间，避免脏数据
                .build(stringRedisTemplate::hasKey); // 查询不到时从 Redis 加载
    }

    /**
     * 字符串 类型缓存
     *
     * @return {@link LoadingCache }<{@link String }, {@link String }>
     */
    @Bean
    public LoadingCache<String, String> stringConfigCache() {
        return Caffeine.newBuilder()
                .expireAfterWrite(60, TimeUnit.SECONDS)
                .maximumSize(1000)
                .build(key -> ConfigContext.me().getConfigValue(key, String.class));
    }


    /**
     * long 类型缓存缓存
     *
     * @return {@link LoadingCache }<{@link String }, {@link Long }>
     */
    @Bean
    public LoadingCache<String, Long> longConfigCache() {
        return Caffeine.newBuilder()
                .expireAfterWrite(5, TimeUnit.MINUTES)
                .maximumSize(1000)
                .build(key -> ConfigContext.me().getConfigValue(key, Long.class));
    }
}
```

**具体使用**

```java
@Resource
private LoadingCache<String, Boolean> tokenExistCache;

String tokenKey = "LOGGED_TOKEN_" + token.toUpperCase(Locale.ROOT);
Boolean hasKey = tokenExistCache.get(tokenKey);
```

