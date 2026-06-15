# MyBatis 面试题

本文件可**单独阅读**，每题含完整参考答案。

---

### 1. MyBatis 是什么？和 Hibernate 区别？

MyBatis 是**半自动 ORM**，SQL 由开发者编写，映射结果到 Java 对象。Hibernate 全自动 ORM，HQL 操作对象，SQL 自动生成。MyBatis SQL 可控、易优化、学习曲线低；Hibernate 开发快、对象模型强、复杂 SQL 难调。

---

### 2. #{} 和 ${} 区别？

**#{}** 预编译占位符，PreparedStatement 参数绑定，**防 SQL 注入**，用于值。

**${}** 字符串直接替换到 SQL，用于**动态表名、列名、ORDER BY** 等，必须白名单校验，不能接用户输入值。

---

### 3. MyBatis 执行流程？

Mapper 接口 → JDK **动态代理** → MapperProxy → SqlSession → Executor → StatementHandler → JDBC → ResultSetHandler 映射结果 → 返回对象。

---

### 4. 一级缓存和二级缓存？

**一级**：SqlSession 级别，默认开启，同一 Session 相同查询走缓存；update/commit/close 清空。跨 Session 不共享。

**二级**：Mapper namespace 级别，需配置 `<cache/>`，跨 Session，存 Serialize 对象，**生产慎用**（脏读、分布式不一致），多用 Redis。

---

### 5. 一级缓存脏读场景？

同一 Session 内 A 改数据未 commit，B 查询可能读到旧缓存；或 commit 后未 clearCache。一般一个 Service 方法一个事务一个 Session，影响可控。

---

### 6. 插件 Interceptor 原理？

实现 Interceptor，@Intercepts 签名拦截 Executor、StatementHandler、ParameterHandler、ResultSetHandler 的 method。责任链包装，PageHelper 分页即拦截 Executor.query 改 SQL 加 limit。

---

### 7. PageHelper 分页原理？

ThreadLocal 存 pageNum/pageSize，拦截 SQL 改写为 count + 分页查询，注意**只对紧跟其后的第一个查询**生效，用完 clearPage。

---

### 8. 延迟加载？

association/collection 设置 fetchType=lazy，首次访问关联对象再查 SQL。可能 **N+1 问题**，应用 join 或批量查询优化。

---

### 9. N+1 是什么？怎么解决？

查 N 条主记录，每条再查关联各 1 次 SQL，共 N+1 次。解决：join 一次查出、嵌套结果映射、@Many 批量 select、业务层 IN 查询。

---

### 10. Mapper 接口没实现类怎么运行？

MyBatis 注册 Mapper 接口，**MapperProxy 动态代理** invoke 方法，根据接口全限定名+方法名找 MappedStatement 执行 SQL。

---

### 11. resultType 和 resultMap？

resultType 直接映射列名到属性（驼峰需 mapUnderscoreToCamelCase）。resultMap 复杂映射：嵌套、discriminator、collection、association。

---

### 12. 动态 SQL 常用标签？

if、choose/when/otherwise、trim、where、set、foreach（IN 列表、批量 insert）。

---

### 13. MyBatis 如何防 SQL 注入？

值用 **#{}**；${} 仅用于受控的结构（表名来自配置而非用户）。like 查询用 `name like concat('%', #{name}, '%')` 而非 ${name}。

---

### 14. MyBatis 和 JDBC 比优势？

免重复资源管理、映射自动化、动态 SQL、插件扩展、Mapper 接口简化 DAO 层。

---

### 15. 事务谁管？

MyBatis 不管理事务，由 **Spring @Transactional** 或编程式事务管理 Connection 提交回滚。SqlSessionFactory 与 DataSourceTransactionManager 配合。

---

### 16. MyBatis-Plus 是什么？

MyBatis 增强：BaseMapper CRUD、Wrapper 条件构造、分页插件、乐观锁 @Version、自动填充、逻辑删除。SQL 仍可手写 XML。

---

### 17. 乐观锁怎么实现？

表加 version 字段，update set ... version=version+1 where id=? and version=?，影响行数 0 则冲突重试或提示。

---

### 18. 逻辑删除？

deleted 字段 0/1，MP @TableLogic 自动 where deleted=0，delete 变 update。唯一索引需考虑 deleted 组合。

---

### 19. 批量插入优化？

foreach 拼 insert values；或 JDBC batch；或 MP saveBatch 分批。注意 SQL 长度和 max_allowed_packet。

---

### 20. 如何打印 SQL 日志？

logging.level.com.xxx.mapper=DEBUG 或 MyBatis logImpl=StdOutImpl（开发）。生产用 DEBUG 谨慎，可采样。

---

### 21. TypeHandler 作用？

Java 类型与 JDBC 类型转换，如 LocalDateTime、JSON 字段存 String 转对象。自定义 @MappedTypes @MappedJdbcTypes。

---

### 22. 一对多 resultMap 怎么写？

collection 标签 ofType，select 嵌套或 join 一次查出用 nested resultMap。

---

### 23. MyBatis 二级缓存为什么不推荐分布式？

各节点本地缓存不互通，更新后其他节点仍可能读旧数据，需 Redis 等集中缓存 + 失效策略。

---

### 24. SqlSessionFactory 和 SqlSession？

Factory 重量级单例，创建 SqlSession。SqlSession 非线程安全，**不要**注入 Singleton Bean，应 Request/方法级或 Mapper 代理内部管理。

---

### 25. 如何做多数据源？

多个 DataSource、SqlSessionFactory、@MapperScan 指定不同 factoryBeanName，@Transactional 指定 transactionManager。

---

### 26. XML 和注解 SQL 选型？

复杂 SQL、动态 SQL 用 XML 清晰。简单 CRUD 注解 @Select 即可。团队统一规范即可。

---

### 27. 为什么 Mapper 方法参数多个要 @Param？

除单个参数外，Java 编译后参数名可能丢失，XML 中 #{name} 需 @Param("name") 或 arg0/param0。

---

### 28. MyBatis 执行器类型？

SIMPLE 普通；REUSE 复用 Statement；BATCH 批量更新，需 flushStatements。

---

### 29. 如何排查 MyBatis 慢 SQL？

Druid 监控、P6Spy、日志耗时、APM 链路、explain 分析索引。

---

### 30. #{} 处理 null？

jdbcType 指定 OTHER/NULL，或 settings jdbcTypeForNull=NULL，避免部分数据库报错。
