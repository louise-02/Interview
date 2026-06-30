Spring Boot 启动过程中 **扩展点执行顺序**、**单 Bean 初始化在链路中的位置**，以及项目里常用的扩展示例（可抄）。

Bean 容器内 9 步生命周期见 [Spring/4、Bean 生命周期与作用域](../../4、Spring/4、Bean生命周期与作用域.md)。

Web 层 `HandlerMethodArgumentResolver`、`ResponseBodyAdvice` 见 [2、拦截器与参数解析](./2、拦截器与参数解析.md)。

---

# 1、Spring Boot 启动全链路（简版）

```
main → SpringApplication.run
  → 启动前：EnvironmentPostProcessor、ApplicationListener(Starting/EnvironmentPrepared)
  → 创建 Context：ApplicationContextInitializer
  → refresh 容器：
        BeanDefinitionRegistryPostProcessor / BeanFactoryPostProcessor
        → 实例化 Bean → 属性注入 → Aware → BPP → 初始化 → BPP
  → 容器刷新完成：ContextRefreshedEvent
  → 启动 Web：ServletWebServerInitializedEvent
  → ApplicationStartedEvent
  → CommandLineRunner / ApplicationRunner
  → ApplicationReadyEvent（应用可对外服务）
  → 关闭：ContextClosedEvent → destroy
```

---

# 2、扩展点执行顺序（对照表）

按 **Boot 2.x Web 应用** 典型顺序排列（与 Spring 版本略有差异，面试/排查按「谁先谁后」理解即可）。

## 2.1 启动阶段（refresh 之前 / 早期）

| 顺序 | 扩展点 | 说明 |
| ---- | ------ | ---- |
| 1 | `ApplicationListener` ← `ApplicationStartingEvent` | 最早之一 |
| 2 | `SpringApplicationRunListener#starting` | 启动监听器 |
| 3 | `EnvironmentPostProcessor#postProcessEnvironment` | 改 Environment，早于 Bean |
| 4 | `ApplicationListener` ← `ApplicationEnvironmentPreparedEvent` | 环境就绪 |
| 5 | `SpringApplicationRunListener#environmentPrepared` | |
| 6 | `ApplicationContextInitializer#initialize` | **Context 刷新前**；不要依赖其他 Bean |
| 7 | `ApplicationListener` ← `ApplicationContextInitializedEvent` | |
| 8 | `SpringApplicationRunListener#contextPrepared` | |
| 9 | `ApplicationListener` ← `ApplicationPreparedEvent` | |
| 10 | `SpringApplicationRunListener#contextLoaded` | |

## 2.2 容器 refresh（BeanDefinition → Bean 实例）

| 顺序 | 扩展点 | 说明 |
| ---- | ------ | ---- |
| 11 | `BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry` | 增删改 BeanDefinition |
| 12 | `BeanDefinitionRegistryPostProcessor#postProcessBeanFactory` | 子接口自带 |
| 13 | `BeanFactoryPostProcessor#postProcessBeanFactory` | **实例化前**改 BeanDefinition |
| 14 | `InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation` | 实例化前，可返回代理对象短路 |
| 15 | **实例化 Bean（构造器）** | |
| 16 | `InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation` | 返回 false 可跳过属性注入 |
| 17 | `InstantiationAwareBeanPostProcessor#postProcessProperties` | 改属性值 |
| 18 | **属性注入** | @Autowired / setter |
| 19 | **Aware** | BeanNameAware、BeanFactoryAware、ApplicationContextAware、EnvironmentAware… |
| 20 | `BeanPostProcessor#postProcessBeforeInitialization` | |
| 21 | `@PostConstruct` | |
| 22 | `MergedBeanDefinitionPostProcessor` 等 | 收集 @Value 等元数据 |
| 23 | `InitializingBean#afterPropertiesSet` | |
| 24 | 自定义 `init-method` | |
| 25 | `BeanPostProcessor#postProcessAfterInitialization` | AOP 代理常在此 |

## 2.3 容器就绪 → 应用可服务

| 顺序 | 扩展点 | 说明 |
| ---- | ------ | ---- |
| 26 | `SmartInitializingSingleton#afterSingletonsInstantiated` | **所有单例 Bean 初始化完成后** |
| 27 | `ApplicationListener` ← `ContextRefreshedEvent` | 容器刷新完成 |
| 28 | `ApplicationListener` ← `ServletWebServerInitializedEvent` | 内嵌 Tomcat 等端口已监听 |
| 29 | `ApplicationListener` ← `ApplicationStartedEvent` | |
| 30 | `SpringApplicationRunListener#started` | |
| 31 | `CommandLineRunner#run` / `ApplicationRunner#run` | **业务初始化常用**（`ApplicationReadyEvent` 之前） |
| 32 | `ApplicationListener` ← `ApplicationReadyEvent` | **最后一环**，可对外提供服务 |
| 33 | `SpringApplicationRunListener#running` | |

## 2.4 关闭

| 顺序 | 扩展点 | 说明 |
| ---- | ------ | ---- |
| 34 | `ApplicationListener` ← `ContextClosedEvent` | 应用退出 |
| 35 | `@PreDestroy` / `DisposableBean#destroy` / `destroy-method` | Bean 销毁 |

### 常用 Boot 生命周期事件

| 事件 | 时机 |
| ---- | ---- |
| `ApplicationStartingEvent` | run 刚开始 |
| `ApplicationEnvironmentPreparedEvent` | Environment 准备好 |
| `ApplicationContextInitializedEvent` | Context 创建后、refresh 前 |
| `ApplicationPreparedEvent` | refresh 前 |
| `ContextRefreshedEvent` | 容器 refresh 完成 |
| `ServletWebServerInitializedEvent` | Web 端口就绪 |
| `ApplicationStartedEvent` | 已 start，Runner 之前 |
| `ApplicationReadyEvent` | **完全就绪** |
| `ContextClosedEvent` | 关闭 |

---

# 3、常用扩展（简例）

## BeanFactoryPostProcessor

**时机**：读取 BeanDefinition 之后、**Bean 实例化之前**。可改 scope、lazy、属性等。

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 修改已注册的 BeanDefinition
    }
}
```

**子接口 `BeanDefinitionRegistryPostProcessor`**：可对 Registry **增删改** BeanDefinition。

```java
void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry);
```

## BeanPostProcessor

**时机**：每个 Bean **初始化前后**。BPP 本身会**优先**初始化；若在 BPP 里注入其他 Bean，可能打乱顺序导致 BPP 失效。

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

| 子接口 | 用途 |
| ------ | ---- |
| `InstantiationAwareBeanPostProcessor` | 实例化前/后、属性注入前；`postProcessAfterInstantiation` 返回 false 可跳过 `postProcessProperties` |
| `SmartInstantiationAwareBeanPostProcessor` | 预测类型、选择构造器、**循环依赖** `getEarlyBeanReference`；AOP 的 `AbstractAutoProxyCreator` |
| `MergedBeanDefinitionPostProcessor` | 收集 @Value、@Autowired 等注入元数据 |

## ApplicationContextInitializer

**时机**：ConfigurableApplicationContext **refresh 之前**；不能依赖已初始化的 Bean。

```java
public class MyApplicationContextInitializer
        implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext ctx) {
        // 设置默认属性、环境变量等
    }
}
```

注册（Boot 2，`META-INF/spring.factories`）：

```properties
org.springframework.context.ApplicationContextInitializer=\
com.example.MyApplicationContextInitializer
```

Boot 3 也可在 `spring.factories` 或 `META-INF/spring/org.springframework.context.ApplicationContextInitializer.imports` 注册（以项目 Boot 版本为准）。

## ApplicationListener / 事件

```java
// 自定义事件
public class MyEvent extends ApplicationEvent {
    public MyEvent(Object source) {
        super(source);
    }
}

// 监听（Bean 方式）
@Component
public class MyEventListener {
    @EventListener
    public void on(MyEvent event) { }
}

// 或实现 ApplicationListener
public class MyEventApplicationListener implements ApplicationListener<MyEvent> {
    @Override
    public void onApplicationEvent(MyEvent event) { }
}
```

**发布事件**（`ApplicationEventPublisherAware`）：

```java
@Component
public class MyPublisher implements ApplicationEventPublisherAware {
    private ApplicationEventPublisher publisher;

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void send() {
        publisher.publishEvent(new MyEvent("hello"));
    }
}
```

**极早监听**（须 SPI，Context 未 refresh 时就要执行）：

```java
public class EnvListener implements ApplicationListener<ApplicationContextInitializedEvent> {
    @Override
    public void onApplicationEvent(ApplicationContextInitializedEvent event) {
        ConfigurableEnvironment env = event.getApplicationContext().getEnvironment();
    }
}
```

```properties
org.springframework.context.ApplicationListener=\
com.example.EnvListener
```

## ApplicationRunner / CommandLineRunner

**时机**：容器就绪、**ApplicationReadyEvent 之前**，适合跑一次性初始化任务。

```java
@Component
public class MyRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // 应用启动后执行
    }
}
```

`CommandLineRunner` 签名 `run(String... args)`，效果类似。

## SpringApplicationRunListener

**时机**：启动各节点统一回调（starting、environmentPrepared、contextPrepared、started、running…）。

```properties
org.springframework.boot.SpringApplicationRunListener=\
com.example.MyRunListener
```

## SmartInitializingSingleton

**时机**：**所有单例 Bean 都初始化完成之后**。

```java
@Component
public class MySmartInit implements SmartInitializingSingleton {
    @Override
    public void afterSingletonsInstantiated() {
        // 依赖其他单例 Bean 的汇总初始化
    }
}
```

## Web 扩展（Mvc）

| 接口 | 作用 |
| ---- | ---- |
| `HandlerMethodArgumentResolver` | 自定义 Controller **入参**解析 |
| `ResponseBodyAdvice` | 统一包装 **返回值** / 改 ResponseBody |

详见 [2、拦截器与参数解析](./2、拦截器与参数解析.md)。

---

# 4、启动后扫描所有 URL

```java
Set<String> urlSet = new HashSet<>();
Map<String, RequestMappingHandlerMapping> mappingMap =
        applicationContext.getBeansOfType(RequestMappingHandlerMapping.class);
for (RequestMappingHandlerMapping mapping : mappingMap.values()) {
    mapping.getHandlerMethods().forEach((info, method) -> {
        if (info.getPatternsCondition() != null) {
            urlSet.addAll(info.getPatternsCondition().getPatterns());
        }
        // Boot 3 / PathPattern 需用 getPathPatternsCondition()，按版本调整
    });
}
```

可用于权限扫描、接口文档导出、网关路由核对。

---

# 5、注意

| 点 | 说明 |
| -- | ---- |
| BPP 里注入 Bean | 易破坏 BPP 注册顺序，尽量无依赖或 `@Lazy` |
| Initializer / 早期 Listener | Context 未 refresh，**不要** `@Autowired` 业务 Bean |
| Runner vs Ready | 需要「端口已监听、健康检查通过」用 `ApplicationReadyEvent`；一般初始化用 Runner 即可 |
| SPI 注册 | Boot 2 常用 `META-INF/spring.factories`；Boot 3 自动配置改 `AutoConfiguration.imports`，Listener/Initializer 仍查官方文档 |
| 循环依赖 | 构造器注入无法解决；setter/字段 + `getEarlyBeanReference`（三级缓存） |
