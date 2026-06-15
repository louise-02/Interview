把公共能力封装成 Starter，业务项目引入依赖即可自动装配。原理见 [2、配置与 Starter](../2、配置与Starter.md#4自定义-starter步骤)。

---

# 1、工程结构（两模块）

```
demo-spring-boot-starter/
├── pom.xml
├── demo-spring-boot-autoconfigure/     # 自动配置实现
│   ├── pom.xml
│   └── src/main/java/.../demo/
│       ├── DemoProperties.java
│       ├── DemoService.java
│       └── DemoAutoConfiguration.java
└── demo-spring-boot-starter/             # 空壳，只聚合依赖
    └── pom.xml
```

业务项目只需引入 `demo-spring-boot-starter`。

---

# 2、父 POM

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>demo-spring-boot-starter-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>demo-spring-boot-autoconfigure</module>
        <module>demo-spring-boot-starter</module>
    </modules>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.18</version>
    </parent>

    <properties>
        <java.version>11</java.version>
    </properties>
</project>
```

---

# 3、autoconfigure 模块

## pom.xml

```xml
<artifactId>demo-spring-boot-autoconfigure</artifactId>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

## DemoProperties

```java
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "demo")
public class DemoProperties {

    /** 是否启用，默认 true */
    private boolean enabled = true;

    /** 问候语前缀 */
    private String prefix = "[Demo]";

    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    public String getPrefix() { return prefix; }
    public void setPrefix(String prefix) { this.prefix = prefix; }
}
```

## DemoService（Starter 对外能力）

```java
public class DemoService {

    private final DemoProperties properties;

    public DemoService(DemoProperties properties) {
        this.properties = properties;
    }

    public String greet(String name) {
        return properties.getPrefix() + " Hello, " + name;
    }
}
```

## DemoAutoConfiguration

```java
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnClass(DemoService.class)
@EnableConfigurationProperties(DemoProperties.class)
@ConditionalOnProperty(prefix = "demo", name = "enabled", havingValue = "true", matchIfMissing = true)
public class DemoAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DemoService demoService(DemoProperties properties) {
        return new DemoService(properties);
    }
}
```

> **知识点**
>
> - `@ConditionalOnMissingBean`：业务项目自己定义 `DemoService` 时，自动配置不生效（可覆盖）。
> - `@ConditionalOnProperty`：`demo.enabled=false` 时整个 Starter 关闭。
> - Boot 2.7+ 用 `@AutoConfiguration`；2.6 及以前用 `@Configuration`。

## 注册自动配置（Boot 2.7+）

文件：`src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```text
com.example.demo.DemoAutoConfiguration
```

Boot 2.6 及以前改用 `META-INF/spring.factories`：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.demo.DemoAutoConfiguration
```

## 配置提示（可选）

`src/main/resources/META-INF/additional-spring-configuration-metadata.json`：

```json
{
  "properties": [
    {
      "name": "demo.prefix",
      "type": "java.lang.String",
      "description": "问候语前缀",
      "defaultValue": "[Demo]"
    },
    {
      "name": "demo.enabled",
      "type": "java.lang.Boolean",
      "description": "是否启用 Demo Starter",
      "defaultValue": true
    }
  ]
}
```

IDEA 写 `application.yml` 时会有补全提示。

---

# 4、starter 空壳模块

## pom.xml

```xml
<artifactId>demo-spring-boot-starter</artifactId>

<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>demo-spring-boot-autoconfigure</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```

只聚合 autoconfigure，**不写 Java 代码**。若 Starter 还需传递第三方库（如 OkHttp），在这里加 dependency。

---

# 5、业务项目使用

## 引入（本地 install 后）

```bash
cd demo-spring-boot-starter
mvn clean install
```

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>demo-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

## application.yml

```yaml
demo:
  prefix: "[MyApp]"
  enabled: true
```

## 注入使用

```java
@RestController
public class HelloController {

    @Autowired
    private DemoService demoService;

    @GetMapping("/hello")
    public String hello() {
        return demoService.greet("World");
    }
}
```

访问 `GET /hello` → `[MyApp] Hello, World`。

---

# 6、验证自动配置是否生效

启动日志加 debug：

```yaml
logging:
  level:
    org.springframework.boot.autoconfigure: DEBUG
```

搜索 `DemoAutoConfiguration` 是否 `matched`。

或：

```bash
curl http://127.0.0.1:8080/actuator/conditions
```

需引入 `spring-boot-starter-actuator` 并暴露 `conditions` 端点。

---

# 7、注意

| 点 | 说明 |
| -- | ---- |
| 命名规范 | 官方 `spring-boot-starter-xxx`；第三方 `xxx-spring-boot-starter` |
| 少依赖 | autoconfigure 只引必要包，避免把 web 强塞给所有用户 |
| 版本 | Starter 的 Boot 版本与业务项目对齐 |
| 发布 | 公司内部私服 `mvn deploy`；开源走 Maven Central |
