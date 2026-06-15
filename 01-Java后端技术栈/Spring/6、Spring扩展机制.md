# 6、Spring 扩展机制

## 1、BeanFactoryPostProcessor

在 Bean **实例化之前**修改 BeanDefinition。

典型：`PropertySourcesPlaceholderConfigurer` 解析 `${}`；`ConfigurationClassPostProcessor` 处理 @Configuration。

可自定义修改 bean 的 scope、属性、是否 lazy。

## 2、BeanPostProcessor

每个 Bean **初始化前后**回调。

- `postProcessBeforeInitialization`
- `postProcessAfterInitialization`（**AOP 代理对象常在此生成**）

@Autowired、@PostConstruct、ApplicationContextAware 等均通过各类 BPP 实现。

## 3、ApplicationListener / @EventListener

观察者模式，解耦业务：

```java
@EventListener
public void onOrderCreated(OrderCreatedEvent event) { }
```

异步：`@Async` + `@EnableAsync`。注意事务边界：异步方法在新线程，不在原事务内。

## 4、Aware 接口

Bean 获取容器基础设施：BeanNameAware、BeanFactoryAware、**ApplicationContextAware**、EnvironmentAware 等。

## 5、FactoryBean

getObject() 返回的对象才是容器中的 Bean，getObjectType、isSingleton。用于复杂对象创建（SqlSessionFactoryBean、ProxyFactoryBean）。

## 6、@Import 扩展

| 方式 | 用途 |
| ---- | ---- |
| 导入 @Configuration | 组合配置 |
| ImportSelector | 根据条件动态选择配置类 |
| ImportBeanDefinitionRegistrar | 编程式注册 BeanDefinition |

Boot 自动配置、SpringFactoriesLoader 大量依赖此机制。

## 7、Spring SPI：spring.factories

Boot 2：`META-INF/spring.factories` 列出 EnableAutoConfiguration、ApplicationListener 等。

Boot 3：AutoConfiguration.imports、其他仍可能用 factories。

## 8、Resource 抽象

ClassPathResource、FileSystemResource、UrlResource 统一访问资源；@Value("classpath:xxx") 注入。

## 9、Environment 与 Profile

`environment.getActiveProfiles()`；@Profile("prod") 条件注册 Bean；PropertySource 链式覆盖。

## 10、设计模式在 Spring 中的体现

| 模式 | 体现 |
| ---- | ---- |
| 工厂 | BeanFactory |
| 单例 | 默认 scope |
| 代理 | AOP、事务 |
| 模板 | JdbcTemplate、RestTemplate |
| 观察者 | ApplicationEvent |
| 适配器 | HandlerAdapter |
| 装饰 | BeanWrapper |
