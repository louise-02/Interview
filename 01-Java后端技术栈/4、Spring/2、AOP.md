# 2、AOP

## 1、AOP 概念

**AOP（面向切面编程）**：将**横切关注点**（日志、事务、权限、监控）从业务代码中剥离，通过**切面**统一织入。

| 概念 | 说明 |
| ---- | ---- |
| **Aspect** | 切面 = 通知 + 切点 |
| **Join Point** | 可织入的点（方法执行、异常等） |
| **Pointcut** | 匹配哪些 Join Point |
| **Advice** | 何时织入：Before、After、Around、AfterReturning、AfterThrowing |
| **Weaving** | 织入方式：编译期、类加载期、**运行期（Spring 默认）** |

## 2、Spring AOP 实现

Spring AOP 默认 **Runtime 织入**，基于**代理**：

| 目标对象 | 代理方式 |
| -------- | -------- |
| 实现了接口 | **JDK 动态代理**（Proxy + InvocationHandler） |
| 无接口 | **CGLIB** 子类代理（不能代理 final 类/方法） |

`proxy-target-class: true`（Boot 2.x+ 默认）→ 优先 CGLIB。

## 3、常用注解

```java
@Aspect
@Component
public class LogAspect {
    @Pointcut("execution(* com.example.service..*(..))")
    public void serviceLayer() {}

    @Around("serviceLayer()")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            log.info("{} cost {}ms", pjp.getSignature(), System.currentTimeMillis() - start);
        }
    }
}
```

**切点表达式**

- `execution(* com.pkg.Service.*(..))` 最常用
- `@annotation(com.example.Log)` 注解切点
- `within(com.example.service..*)` 包内

## 4、AOP 与事务

`@Transactional` 本质是 **TransactionInterceptor** + AOP，对 public 方法织入事务边界。

**同类内部调用失效**：`this.save()` 不经过代理 → 需注入自身或 `AopContext.currentProxy()`。

## 5、AspectJ vs Spring AOP

| | Spring AOP | AspectJ |
| - | ---------- | ------- |
| 织入 | 仅方法（代理） | 字段、构造器、静态等 |
| 性能 | 运行时代理 | 编译/加载期织入，更强 |

## 6、常见面试题

**1. JDK 动态代理和 CGLIB 区别？**

JDK 要接口；CGLIB 子类，不能代理 final；Spring 可配置优先 CGLIB。

**2. 为什么 @Transactional 同类调用不生效？**

调用未走代理对象，事务 Advice 未织入。

**3. Around 和 Before 区别？**

Around 可控制是否 `proceed()`、改返回值；Before 不能在方法后统一处理（除非配 After）。
