# 1、前置

**Redis 连接与 RedisTemplate** 先按 [16、Redis集成](./16、Redis集成.md) 配好。本章只讲 **Spring Cache 声明式缓存**（`@Cacheable` 等）。

额外依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

---

# 2、application.yml

在 [16、Redis集成](./16、Redis集成.md) 的 `spring.data.redis` 基础上，增加：

```yaml
spring:
  cache:
    type: redis                        # 声明 Cache 走 Redis；单机可改 caffeine
    redis:
      time-to-live: 30m                # 默认 entry TTL（可按 cacheName 再定制）
      cache-null-values: false         # 不缓存 null，防穿透
      key-prefix: "app:cache:"         # 可选，Key 前缀，便于 Redis 里区分
      use-key-prefix: true
```

**Caffeine 本地缓存**（不用 Redis 时）：

```yaml
spring:
  cache:
    type: caffeine
  caffeine:
    spec: maximumSize=1000,expireAfterWrite=30m
```

| | Redis Cache | Caffeine |
| - | ----------- | -------- |
| 场景 | 多实例共享 | 单机 JVM |
| 依赖 | 16 + cache starter | cache + caffeine |

---

# 3、开启缓存

```java
@Configuration
@EnableCaching   // 启用 @Cacheable / @CacheEvict / @CachePut
public class SpringCacheConfig {
}
```

---

# 4、常用注解

| 注解 | 作用 |
| ---- | ---- |
| `@Cacheable` | 命中缓存则不调方法；否则执行并写入 |
| `@CacheEvict` | 删缓存；`allEntries = true` 清整个 cacheName |
| `@CachePut` | 始终执行方法并更新缓存 |
| `@Caching` | 组合多个操作 |

```java
@Cacheable(value = "user", key = "#id", unless = "#result == null")
public User getById(Long id) { ... }

@CacheEvict(value = "user", key = "#id")
public void updateUser(Long id, UserDTO dto) { ... }
```

**同类内部自调用不生效**（无 AOP 代理），需拆 Service 或注入自身。

---

# 5、缓存对象与 Redis 存什么

`spring.cache.type=redis` 时，值序列化走 Spring Cache 的 Redis 配置（默认 JDK 序列化）。生产可统一 JSON：

```java
@Bean
RedisCacheConfiguration redisCacheConfiguration() {
    return RedisCacheConfiguration.defaultCacheConfig()
        .serializeKeysWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new StringRedisSerializer()))
        .serializeValuesWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new GenericJackson2JsonRedisSerializer()))
        .entryTtl(Duration.ofMinutes(30))
        .disableCachingNullValues();
}

@Bean
RedisCacheManager cacheManager(RedisConnectionFactory factory,
                               RedisCacheConfiguration config) {
    return RedisCacheManager.builder(factory)
        .cacheDefaults(config)
        .build();
}
```

或实体 **`implements Serializable`**（简单场景够用）。

---

# 6、更新数据时删缓存

改库后必须 evict，否则最长等到 TTL：

```java
@CacheEvict(value = "userDetails", key = "#username")
public void updateUserRoles(String username, List<Long> roleIds) {
    // 写库 ...
}

@Resource
private CacheManager cacheManager;

public void evict(String username) {
    Cache cache = cacheManager.getCache("userDetails");
    if (cache != null) {
        cache.evict(username);
    }
}
```

---

# 7、典型场景：Security UserDetails

JWT 只带 username 时，每个请求会 `loadUserByUsername` 查库。高 QPS 时：

```java
@Override
@Cacheable(value = "userDetails", key = "#username", unless = "#result == null")
public UserDetails loadUserByUsername(String username) {
    // 查用户 + 角色/权限 ...
}
```

**何时 evict**：改角色、改权限、禁用用户、改密码。  
Security 上下文见 [SpringSecurity/0、生产集成完整方案 §6.3](../../8、SpringSecurity/0、生产集成完整方案.md)。

---

# 8、踩坑

| 现象 | 原因 |
| ---- | ---- |
| 注解不生效 | 未 `@EnableCaching`；方法非 public；同类自调用 |
| 改库后仍是旧数据 | 未 `@CacheEvict` |
| 多实例缓存不一致 | 用了 Caffeine 或未共用 Redis |
| Redis 乱码 / 反序列化失败 | 序列化方式与存时不一致 |
| 与手写 RedisTemplate Key 冲突 | Cache 有 `key-prefix`，注意前缀规划 |

Redis 连接、连接池、RedisTemplate 见 [16、Redis集成](./16、Redis集成.md)。
