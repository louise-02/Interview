# 1、IoC 与依赖注入

## 1、IoC 与 DI

**IoC（控制反转）**：对象创建与依赖关系由**容器**管理，而不是对象自己 `new`。

**DI（依赖注入）**：容器通过构造器、Setter、字段等方式把依赖**注入**到对象中。

```
传统：A 内部 new B()  →  A 依赖 B 的实现，难测试
Spring：容器创建 B，注入 A  →  A 只依赖接口，可 Mock
```

## 2、Spring 容器

| 接口 | 实现 | 说明 |
| ---- | ---- | ---- |
| `BeanFactory` | `DefaultListableBeanFactory` | 基础容器 |
| `ApplicationContext` | `AnnotationConfigApplicationContext`、`ClassPathXmlApplicationContext` | 企业级，事件、国际化、AOP 等 |

Boot 中本质是 **AnnotationConfigServletWebServerApplicationContext**。

## 3、Bean 的定义方式

| 方式 | 示例 |
| ---- | ---- |
| `@Component` 及衍生 | `@Service`、`@Repository`、`@Controller` |
| `@Bean` | `@Configuration` 类中的工厂方法 |
| XML | `<bean id="" class=""/>`（旧项目） |
| `@Import` | 导入配置类或普通类 |

**组件扫描**：`@ComponentScan(basePackages = "com.example")`。

## 4、依赖注入方式

| 方式 | 推荐 | 说明 |
| ---- | ---- | ---- |
| **构造器注入** | ✅ 推荐 | 依赖 final、易测、避免半初始化 |
| **Setter 注入** | 可选 | 可选依赖 |
| **字段 @Autowired** | 不推荐 | 简洁但难测、隐藏依赖 |

```java
@Service
public class OrderService {
    private final OrderMapper orderMapper;

    public OrderService(OrderMapper orderMapper) {
        this.orderMapper = orderMapper;
    }
}
```

`@Autowired` + `@Qualifier("beanName")` 解决多实现；`@Primary` 指定默认 Bean。

## 5、@Autowired 与 @Resource

| 注解 | 来源 | 匹配规则 |
| ---- | ---- | -------- |
| `@Autowired` | Spring | 按类型；多个同类型按名称 |
| `@Resource` | JSR-250 | 先名称后类型 |
| `@Inject` | JSR-330 | 同 Autowired |

## 6、循环依赖（三级缓存）

仅 **单例 + 字段/Setter 注入** 可通过三级缓存解决；**构造器循环依赖** 启动失败。

```
singletonObjects          → 完整 Bean
earlySingletonObjects     → 早期引用（未完成属性填充）
singletonFactories        → ObjectFactory，用于 AOP 代理早期暴露
```

**流程简述**：A 创建中需要 B → B 创建中需要 A → 从工厂拿 A 的早期引用给 B → B 完成 → A 完成。

**prototype** 作用域不缓存，无法三级缓存解决循环依赖。

## 7、常见面试题

**1. IoC 和 DI 区别？**

IoC 是思想（控制权反转）；DI 是实现手段（注入依赖）。

**2. 为什么推荐构造器注入？**

依赖不可变、必填、便于单元测试、避免循环依赖在构造阶段暴露。

**3. Spring 如何解决循环依赖？**

单例 Bean 三级缓存 + 提前暴露早期引用；构造器注入不行。
