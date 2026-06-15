# Spring 面试题

本文件可**单独阅读**，每题含完整参考答案。

---

### 1. 什么是 IoC 和 DI？

IoC（控制反转）：对象的创建和依赖关系由 **Spring 容器**管理，而不是在类内部 new。DI（依赖注入）是 IoC 的实现方式：容器通过构造器、Setter 或字段把依赖注入进来。好处：解耦、易测试（可 Mock）、便于替换实现。

---

### 2. Spring Bean 有哪些作用域？

**singleton**（默认，容器内一个）、**prototype**（每次 getBean 新建）、**request**、**session**、**application**（Web）。singleton Bean 若持有状态需考虑线程安全；有状态 Bean 常用 prototype。

---

### 3. @Autowired 怎么注入？推荐哪种？

按类型匹配，多个同类型按名称或 @Qualifier/@Primary。推荐 **构造器注入**：依赖 final、必填、易单元测试、避免未完全初始化。字段注入简洁但不推荐生产代码。

---

### 4. @Resource 和 @Autowired 区别？

@Autowired Spring 注解，先类型后名称。@Resource JSR-250，先名称后类型。两者都可注入，Spring 项目常用 @Autowired + 构造器。

---

### 5. Spring 如何解决循环依赖？

仅 **单例 + 字段/Setter 注入**：三级缓存（singletonObjects、earlySingletonObjects、singletonFactories）提前暴露早期引用。**构造器循环依赖**无法解决，启动报错。prototype 不缓存，也无法三级缓存解决。

---

### 6. 三级缓存分别是什么？

① 完整单例池；② 早期引用池（未完成属性填充）；③ ObjectFactory 工厂，用于需要 AOP 代理时暴露早期代理对象。解决 A 依赖 B、B 依赖 A 的单例创建。

---

### 7. Spring AOP 原理？

运行期 **动态代理**。目标类有接口常用 **JDK 动态代理**（Proxy + InvocationHandler）；无接口用 **CGLIB** 子类代理（不能代理 final 类/方法）。Boot 2.x+ 默认 `proxy-target-class=true` 优先 CGLIB。

---

### 8. AOP 有哪些 Advice 类型？

Before、After、AfterReturning、AfterThrowing、**Around**（可控制是否 proceed，最强大）。切点 Pointcut 匹配 Join Point，Aspect = 切点 + 通知。

---

### 9. JDK 动态代理和 CGLIB 区别？

JDK 必须有接口，基于反射调用接口方法。CGLIB 生成子类覆盖方法，不能代理 final/static。JDK 创建快；CGLIB 首次慢但调用可略快。Spring 根据配置和目标类选择。

---

### 10. @Transactional 原理？

基于 **AOP**，`TransactionInterceptor` 在方法前后开启/提交/回滚 JDBC 事务（PlatformTransactionManager）。代理对象执行才生效。

---

### 11. 事务传播 REQUIRED 和 REQUIRES_NEW？

**REQUIRED**（默认）：当前有事务则加入，无则新建，共用一个事务，一起回滚。

**REQUIRES_NEW**：总是新建事务，**挂起**外层，内外独立提交/回滚。适合日志、审计必须落库的场景。

---

### 12. NESTED 和 REQUIRES_NEW 区别？

NESTED 用 **Savepoint** 嵌套在外层事务内，外层回滚则内层也回滚；内层回滚可只回滚到 savepoint。REQUIRES_NEW 完全独立连接/事务。

---

### 13. @Transactional 什么时候失效？

① 方法非 public；② **同类 this 调用**未走代理；③ 异常被 catch 吞掉；④ **checked Exception** 默认不回滚；⑤ 类不是 Spring Bean；⑥ 数据库引擎不支持（MyISAM）。

---

### 14. 同类调用事务怎么解决？

注入自身接口代理、`AopContext.currentProxy()`（需 `@EnableAspectJAutoProxy(exposeProxy=true)`）、拆到另一个 Bean、改用 AspectJ 编译期织入（少见）。

---

### 15. Bean 生命周期主要步骤？

实例化 → 属性注入 → Aware 回调 → BeanPostProcessor.before → @PostConstruct/InitializingBean → BeanPostProcessor.after（**AOP 代理常在此生成**）→ 使用 → @PreDestroy/destroy 销毁。

---

### 16. BeanFactory 和 ApplicationContext？

ApplicationContext 是 BeanFactory 子接口，增加：自动 BeanPostProcessor/BeanFactoryPostProcessor 注册、国际化、事件发布、Environment 等。企业开发都用 ApplicationContext。

---

### 17. @Configuration 和 @Component 区别？

@Configuration 类中 @Bean 方法会被 **CGLIB 增强**，类内 `@Bean` 方法互相调用仍返回**同一单例**。普通 @Component 中 @Bean 直接调用等于多次 new。

---

### 18. @Component @Service @Repository @Controller 区别？

都是 @Component 衍生，语义区分：服务层、DAO 层、控制器层。功能相同，便于分层和 AOP/异常转换（PersistenceExceptionTranslation）。

---

### 19. Spring 事件机制？

ApplicationEvent + ApplicationListener，或 `@EventListener`。同步默认在同线程；可 `@Async` 异步。用于解耦（订单创建后发通知）。

---

### 20. BeanFactoryPostProcessor 做什么？

在 Bean 实例化**之前**修改 BeanDefinition，如 PropertySourcesPlaceholderConfigurer 解析 ${} 占位符。

---

### 21. BeanPostProcessor 做什么？

每个 Bean 初始化前后回调。AOP、@Autowired 解析、@PostConstruct 等都依赖各类 BeanPostProcessor。

---

### 22. 如何自定义 Scope？

实现 Scope 接口，registerScope，如自定义 thread scope（注意清理）。

---

### 23. Spring 单例 Bean 线程安全吗？

Bean 本身无状态则安全（Service 只调 DAO）。若 Bean 有**成员可变状态**则不安全，需 prototype、无状态设计或加锁。

---

### 24. @Lazy 作用？

延迟到首次使用时才创建 Bean，加快启动；注意首次访问仍要初始化，不能掩盖循环依赖设计问题。

---

### 25. @Primary 和 @Qualifier？

多个同类型 Bean 时，@Primary 指定默认；@Qualifier("name") 指定注入哪一个。

---

### 26. FactoryBean 是什么？

实现 FactoryBean 接口，getObject() 返回的实际对象是容器管理的 Bean，常用于复杂对象创建（MyBatis SqlSessionFactoryBean）。

---

### 27. Spring 如何处理静态 @Value？

@Value 可注入 static 字段，需 setter 上 @Value 或 @PostConstruct 静态赋值技巧；更推荐实例字段 + 非 static 方法。

---

### 28. @Conditional 在 Boot 里怎么用？

@ConditionalOnClass、OnMissingBean、OnProperty 等，自动配置类根据 classpath 和配置决定是否注册 Bean。

---

### 29. Spring 事务隔离级别默认？

DEFAULT，使用数据库默认。MySQL InnoDB 默认 **REPEATABLE READ**，可配 READ_COMMITTED 等。

---

### 30. 声明式事务和编程式事务？

声明式 @Transactional 简洁。编程式 TransactionTemplate 手动 execute，适合极小范围或特殊控制，少用。

---

### 31. AspectJ 和 Spring AOP 区别？

Spring AOP 仅方法级别、运行期代理。AspectJ 编译/加载期织入，可字段/构造器/静态，功能更强，Spring 默认不用全 AspectJ。

---

### 32. 为什么推荐构造器注入解决循环依赖问题？

构造器注入无法在对象半完成时注入，**强制**你在设计上去掉循环依赖，比三级缓存掩盖问题更清晰。

---

### 33. @Import 几种用法？

导入 @Configuration 类、ImportSelector 动态选择、ImportBeanDefinitionRegistrar 注册定义。

---

### 34. Environment 能读什么？

系统属性、环境变量、application.properties、Profile 特定配置。`environment.getProperty("key")`。

---

### 35. Spring 设计模式有哪些？

工厂（BeanFactory）、单例、代理（AOP）、模板（JdbcTemplate）、观察者（事件）、适配器（HandlerAdapter）等。
