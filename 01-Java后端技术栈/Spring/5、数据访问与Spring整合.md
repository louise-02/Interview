# 5、数据访问与 Spring 整合

## 1、JdbcTemplate

Spring 对 JDBC 的封装，**模板方法**消除重复：获取连接、创建 Statement、处理异常、关闭资源。

```java
jdbcTemplate.query("select id,name from user where id=?",
    (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name")),
    id);
```

优点：SQL 可控；缺点：样板代码仍多，大项目多用 MyBatis/JPA。

## 2、事务与 DataSource

`DataSourceTransactionManager` 绑定线程 Connection：`@Transactional` 方法内同一 Connection 提交/回滚。

**多数据源**：多个 DataSource + 多个 TransactionManager + `@Transactional(transactionManager="xxx")` + Mapper 指定 SqlSessionFactory。

## 3、Spring 整合 MyBatis

`SqlSessionFactoryBean` 读数据源、mapperLocations、typeAliasesPackage → 生成 `SqlSessionFactory`。

`MapperScannerConfigurer` 或 `@MapperScan` 扫描 Mapper 接口注册 Bean。

事务由 Spring 管理，MyBatis 仅执行 SQL。

## 4、Spring Data JPA（了解）

Repository 接口派生查询、`@Query` JPQL、分页 Pageable。Hibernate 作 JPA 实现。开发快，复杂 SQL 和优化不如 MyBatis 直观。

| | MyBatis | JPA |
| - | ------- | --- |
| SQL | 手写 | 自动生成为主 |
| 学习 | SQL 导向 | 对象模型导向 |
| 场景 | 互联网 CRUD+复杂 SQL | 领域模型、快速 CRUD |

## 5、连接池

Boot 默认 **HikariCP**（快、稳）。Druid 带监控 SQL、wall 防火墙。配置 maximum-pool-size 不宜超过 DB 承载；连接泄漏检测 leakDetectionThreshold。

## 6、@Transactional 与只读

`@Transactional(readOnly = true)` 提示数据库只读优化，**路由到从库**时可配合 AbstractRoutingDataSource 实现读写分离。

## 7、常见面试点

- Spring 声明式事务基于 AOP + TransactionManager。
- 同一类内调用事务方法不生效：未走代理。
- MyBatis 一级缓存 Session 级，Spring 每事务一般一个 SqlSession，注意缓存边界。
