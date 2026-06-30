# 1、依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!-- Lettuce 连接池需要（Boot 3 默认 Lettuce） -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

Boot 自动配置 **`RedisConnectionFactory`**、**`RedisTemplate`**、**`StringRedisTemplate`**（见 `RedisAutoConfiguration`）。

---

# 2、application.yml（单机）

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password:                          # 有密码时填写；生产放配置中心
      database: 0                        # 默认 0；多业务可拆库号
      timeout: 3s                        # 命令超时
      connect-timeout: 3s
      client-type: lettuce               # Boot 2+ 默认 lettuce（旧版 jedis 需换依赖）
      lettuce:
        pool:
          enabled: true
          max-active: 16                 # 最大连接数
          max-idle: 8
          min-idle: 2
          max-wait: 3s                   # 借连接最长等待
```

生产：host/password 走环境变量或 Nacos；`database` 按环境区分（dev=0, prod=1 等）。

---

# 3、集群 / 哨兵（了解）

**哨兵**（主从 + 自动故障转移）：

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes:
          - 10.0.0.1:26379
          - 10.0.0.2:26379
      password: xxx
      lettuce:
        pool:
          enabled: true
          max-active: 16
```

**Cluster**：

```yaml
spring:
  data:
    redis:
      cluster:
        nodes:
          - 10.0.0.1:6379
          - 10.0.0.2:6379
          - 10.0.0.3:6379
      password: xxx
```

---

# 4、RedisTemplate 序列化（必配）

默认 JDK 序列化可读性差、跨语言不友好。生产推荐 **Key 字符串 + Value JSON**：

```java
@Configuration
public class RedisConfig {

    @Bean
    RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        StringRedisSerializer str = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer json = new GenericJackson2JsonRedisSerializer();

        template.setKeySerializer(str);
        template.setHashKeySerializer(str);
        template.setValueSerializer(json);
        template.setHashValueSerializer(json);
        template.afterPropertiesSet();
        return template;
    }
}
```

纯字符串场景可直接注入 **`StringRedisTemplate`**（Key/Value 都是 String），不必再配序列化。

---

# 5、常用操作

```java
@Service
@RequiredArgsConstructor
public class RedisService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final StringRedisTemplate stringRedisTemplate;

    // String
    public void set(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }

    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    // Hash
    public void hSet(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }

    // 过期 / 删除
    public Boolean expire(String key, Duration ttl) {
        return redisTemplate.expire(key, ttl);
    }

    public Boolean delete(String key) {
        return redisTemplate.delete(key);
    }

    // 简单计数
    public Long incr(String key) {
        return stringRedisTemplate.opsForValue().increment(key);
    }
}
```

业务 Key 建议统一前缀：`app:user:1001`，便于排查与按前缀清理。

---

# 6、上线检查

- [ ] 生产密码不进 Git；超时、连接池按 QPS 调过  
- [ ] RedisTemplate 已改 JSON/String 序列化，不用默认 JDK  
- [ ] Key 有业务前缀；敏感数据设 TTL  
- [ ] 多实例共享数据时用 Redis；纯本地临时数据不必上 Redis  

Redis 命令与数据类型原理见 [02-技术栈/Redis](../../../02-技术栈/Redis/Redis.md)。
