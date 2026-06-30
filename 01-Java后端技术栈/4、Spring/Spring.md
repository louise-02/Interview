# Spring Framework

Spring Framework 是 Java 企业级应用的**基础容器**，提供 IoC/DI、AOP、事务、数据访问整合等。Spring Boot / Spring Cloud 都构建在其之上。

## 目录

| 章节 | 内容 |
| ---- | ---- |
| [1、IoC 与依赖注入](./1、IoC与依赖注入.md) | 容器、Bean、注入方式、循环依赖 |
| [2、AOP](./2、AOP.md) | 切面、代理、JDK/CGLIB、应用场景 |
| [3、事务管理](./3、事务管理.md) | 传播行为、隔离级别、失效场景 |
| [4、Bean 生命周期与作用域](./4、Bean生命周期与作用域.md) | 生命周期 9 步、Aware、BPP、Scope、@Configuration |
| [5、数据访问与 Spring 整合](./5、数据访问与Spring整合.md) | JdbcTemplate、MyBatis/JPA、连接池、读写分离 |
| [6、Spring 扩展机制](./6、Spring扩展机制.md) | BFPP、BPP、事件、Import、SPI |

## 版本关系

| 框架 | 与 Spring 关系 |
| ---- | -------------- |
| Spring MVC | Spring Web 模块，MVC 实现 |
| Spring Boot | 自动配置 + Starter，内嵌 Tomcat |
| Spring Cloud | 微服务治理，基于 Boot |

## 学习顺序

Spring IoC → AOP → 事务 → Bean 生命周期 → 再学 SpringMVC / Boot。
