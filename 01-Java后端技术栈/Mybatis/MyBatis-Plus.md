# 简介

官网地址：https://baomidou.com/getting-started/

**添加依赖**

```xml
  <dependency>
      <groupId>com.baomidou</groupId>
      <artifactId>mybatis-plus-boot-starter</artifactId>
      <version>3.5.15</version>
  </dependency>
```

# 1、常用配置

## 1、配置文件

详细配置

```yml
mybatis-plus:
  # =============================================
  # 1. MyBatis-Plus 全局配置 (global-config)
  # =============================================
  global-config:
    # 是否控制台输出 MyBatis-Plus 的 LOGO  banner
    # 默认值: true
    banner: true
    
    # 是否启用 SQL 性能分析插件(3.2.0已移除，使用其他方式)
    # 默认值: false
    enable-sql-runner: false
    
    # 是否在启动时检查配置
    # 默认值: true
    check-config: true
    
    # 全局超时时间(秒)
    # 默认值: null
    timeout: 30
    
    # 数据库相关配置
    db-config:
      # ========== 主键策略配置 ==========
      # 主键类型
      # 可选值:
      #   AUTO:          数据库ID自增
      #   NONE:          无状态，该类型为未设置主键类型(注解里等于跟随全局,全局里约等于 INPUT)
      #   INPUT:         用户输入ID
      #   ASSIGN_ID:     分配ID(主键类型为Number(Long)或String)(默认雪花算法)
      #   ASSIGN_UUID:   分配UUID(主键类型为String)
      # 默认值: ASSIGN_ID
      id-type: ASSIGN_ID
      
      # ID生成器，配合ASSIGN_ID、ASSIGN_UUID使用
      # 默认值: com.baomidou.mybatisplus.core.incrementer.DefaultIdentifierGenerator
      identifier-generator: com.baomidou.mybatisplus.core.incrementer.DefaultIdentifierGenerator
      
      # ========== 字段策略配置 ==========
      # 插入时字段策略
      # 可选值:
      #   IGNORED:       忽略判断，所有字段都插入
      #   NOT_NULL:      非NULL判断，只插入非NULL字段
      #   NOT_EMPTY:     非空判断，对字符串会检查是否为空字符串
      # 默认值: NOT_NULL
      insert-strategy: NOT_NULL
      
      # 更新时字段策略
      # 可选值同insert-strategy
      # 默认值: NOT_NULL
      update-strategy: NOT_NULL
      
      # 查询时字段策略(where条件)
      # 可选值同insert-strategy
      # 默认值: NOT_NULL
      select-strategy: NOT_NULL
      
      # ========== 逻辑删除配置 ==========
      # 逻辑删除字段名
      # 默认值: null
      logic-delete-field: deleted
      
      # 逻辑已删除值
      # 默认值: 1
      logic-delete-value: 1
      
      # 逻辑未删除值
      # 默认值: 0
      logic-not-delete-value: 0
      
      # ========== 表名配置 ==========
      # 表名前缀
      # 默认值: null
      table-prefix: t_
      
      # 表名后缀
      # 默认值: null
      table-suffix: 
      
      # 表名是否使用下划线转驼峰命名
      # 默认值: true
      table-underline: true
      
      # 是否大写命名(如果这里配置为true，表名和字段名都会转换为大写)
      # 默认值: false
      capital-mode: false
      
      # ========== 数据库类型配置 ==========
      # 数据库类型
      # 可选值:
      #   MYSQL, MARIADB, ORACLE, DB2, H2, HSQL, SQLITE, POSTGRE_SQL,
      #   SQL_SERVER2005, SQL_SERVER, DM, KINGBASE_ES, CLICK_HOUSE
      # 默认值: 自动检测
      db-type: MYSQL
      
      # ========== 其他配置 ==========
      # 字段格式化，将表名和列名格式化
      # 默认值: false
      capital-mode: false
      
      # 是否跳过结果集处理
      # 默认值: false
      skip-result-set: false
      
      # 是否启用schema
      # 默认值: false
      enable-schema: false
      
      # 列格式
      # 默认值: null
      column-format: 
      
      # 表格式
      # 默认值: null
      table-format: 

  # =============================================
  # 2. MyBatis 原生配置 (configuration)
  # =============================================
  configuration:
    # ========== 映射配置 ==========
    # 是否开启自动驼峰命名规则映射
    # 数据库字段: user_name → 实体类属性: userName
    # 默认值: true
    map-underscore-to-camel-case: true
    
    # 自动映射行为
    # 可选值:
    #   NONE:   禁用自动映射
    #   PARTIAL: 只自动映射没有定义嵌套结果映射的字段(默认)
    #   FULL:   自动映射所有结果，包括嵌套结果
    # 默认值: PARTIAL
    auto-mapping-behavior: PARTIAL
    
    # 自动映射未知列/属性的处理方式
    # 可选值:
    #   NONE:   不做任何反应 (默认)
    #   WARNING: 输出警告日志
    #   FAILING: 映射失败，抛出异常
    # 默认值: NONE
    auto-mapping-unknown-column-behavior: NONE
    
    # ========== 缓存配置 ==========
    # 是否开启二级缓存
    # 默认值: true
    cache-enabled: true
    
    # 本地缓存范围
    # 可选值:
    #   SESSION:  会话级别缓存
    #   STATEMENT:语句级别缓存(不会缓存)
    # 默认值: SESSION
    local-cache-scope: SESSION
    
    # ========== 结果集配置 ==========
    # 当返回行的所有列都是空时，MyBatis默认返回null。
    # 当开启这个设置时，MyBatis会返回一个空实例。
    # 请注意，它也适用于嵌套的结果集 (3.4.2+)
    # 默认值: false
    return-instance-for-empty-row: false
    
    # 查询时，关闭null值警告
    # 默认值: false
    call-setters-on-nulls: false
    
    # ========== 日志配置 ==========
    # 日志实现
    # 可选值:
    #   org.apache.ibatis.logging.stdout.StdOutImpl        控制台输出
    #   org.apache.ibatis.logging.slf4j.Slf4jImpl          SLF4J
    #   org.apache.ibatis.logging.log4j2.Log4j2Impl        Log4j2
    #   org.apache.ibatis.logging.commons.JakartaCommonsLoggingImpl  Jakarta Commons Logging
    #   org.apache.ibatis.logging.nologging.NoLoggingImpl  无日志
    # 默认值: 未设置
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
    
    # ========== 执行器配置 ==========
    # 默认执行器
    # 可选值:
    #   SIMPLE:  简单执行器，每次执行都会创建一个新的预处理语句
    #   REUSE:   复用执行器，复用预处理语句
    #   BATCH:   批处理执行器，批量执行所有更新语句
    # 默认值: SIMPLE
    default-executor-type: SIMPLE
    
    # 默认语句超时时间(秒)
    # 默认值: null (使用驱动默认)
    default-statement-timeout: 30
    
    # 默认获取数量限制
    # 默认值: null (使用驱动默认)
    default-fetch-size: 100
    
    # ========== 其他配置 ==========
    # 是否使用列标签代替列名
    # 默认值: true
    use-column-label: true
    
    # 是否允许JDBC支持自动生成主键
    # 默认值: false
    use-generated-keys: false
    
    # 是否允许单个语句返回多个结果集
    # 默认值: true
    multiple-result-sets-enabled: true
    
    # 延迟加载的触发方法
    # 默认值: equals,clone,hashCode,toString
    lazy-load-trigger-methods: equals,clone,hashCode,toString
    
    # 是否启用懒加载
    # 默认值: false
    lazy-loading-enabled: false
    
    # aggressive-lazy-loading的开关
    # 默认值: false
    aggressive-lazy-loading: false
    
    # 设置JDBC类型为NULL
    # 默认值: OTHER
    jdbc-type-for-null: null
    
    # 懒加载时属性的加载方式
    # 可选值:
    #   AGGRESSIVE: 积极模式
    #   DEFERRED:   延迟模式
    # 默认值: null
    lazy-loading-force-use-of-constructor: 
    
    # 代理工厂
    # 可选值:
    #   JAVASSIST: Javassist代理工厂
    #   CGLIB:     CGLIB代理工厂
    # 默认值: JAVASSIST
    proxy-factory: JAVASSIST
    
    # 是否启用sharding-jdbc自动路由
    # 默认值: false
    sharding-enable: false
    
    # 是否启用MyBatis的默认枚举类型处理器
    # 默认值: false
    use-actual-param-name: true

  # =============================================
  # 3. MyBatis 映射器配置
  # =============================================
  # MyBatis映射文件的位置
  # 支持Ant风格的通配符
  # 默认值: null
  mapper-locations: classpath*:/mapper/**/*.xml
  
  # 类型别名所在的包
  # 设置后，在Mapper XML中可以直接使用类名，而不需要写全限定名
  # 默认值: null
  type-aliases-package: com.example.entity
  
  # 类型处理器所在的包
  # 默认值: null
  type-handlers-package: com.example.handler
  
  # 检查MyBatis配置文件是否存在
  # 默认值: false
  check-config-location: false
  
  # 类型别名的超类型
  # 只有该超类的子类才会被注册为类型别名
  # 默认值: null
  type-aliases-super-type: java.lang.Object
  
  # 执行器类型
  # 可选值: SIMPLE, REUSE, BATCH
  # 默认值: SIMPLE
  executor-type: SIMPLE
  
  # 配置默认的枚举类型处理器
  # 默认值: org.apache.ibatis.type.EnumTypeHandler
  default-enum-type-handler: org.apache.ibatis.type.EnumTypeHandler
  
  # 配置文件位置
  # 默认值: null
  config-location: classpath:mybatis-config.xml

  # =============================================
  # 4. MyBatis-Plus 扩展配置
  # =============================================
  # 分页插件配置(3.4.0+)
  # 是否启用分页插件
  # 默认值: true
  page:
    enable: true
    
  # 安全模式配置
  # 是否启用安全模式，防止全表更新/删除
  # 默认值: false  
  safe-mode: false
  
  # 元对象handler配置
  # 是否启用自动填充
  # 默认值: true
  meta-object-handler:
    enable: true
```

常用配置

```yml
mybatis-plus:
  global-config:
    db-config:
      id-type: ASSIGN_ID           # 主键策略
      logic-delete-field: deleted  # 逻辑删除
      insert-strategy: NOT_NULL    # 插入策略
      update-strategy: NOT_NULL    # 更新策略
  configuration:
    map-underscore-to-camel-case: true  # 驼峰映射
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl  # SQL日志
  mapper-locations: classpath*:/mapper/**/*.xml  # XML位置
  type-aliases-package: com.louise.entity       # 实体类包
```

## 2、配置类

```java
@Configuration
public class MybatisPlusConfig {
    
    /**
     * 插件
     */
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        
        // 分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        
        // 乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        
        // 防止全表更新与删除插件
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
        
        return interceptor;
    }
    
    /**
     * 自动填充处理器
     */
    @Bean
    public MetaObjectHandler metaObjectHandler() {
        return new MetaObjectHandler() {
            @Override
            public void insertFill(MetaObject metaObject) {
                this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
            }
            
            @Override
            public void updateFill(MetaObject metaObject) {
                this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
            }
        };
    }
}
```



# 2、常用功能

## 1、通用 Mapper

无需编写任何 Mapper XML 文件，通过继承 `BaseMapper` 接口，即可获得强大的 CRUD 方法。

```java
// 1. 让你的 Mapper 接口继承 BaseMapper
public interface UserMapper extends BaseMapper<User> {
    // 无需编写任何方法，就已拥有如下方法：
    // int insert(T entity);
    // int deleteById(Serializable id);
    // int updateById(T entity);
    // T selectById(Serializable id);
    // List<T> selectList(Wrapper<T> queryWrapper);
    // ... 等等
}

// 2. 在Service层可以直接使用
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    public void someMethod() {
        User user = new User();
        user.setName("张三");
        user.setAge(28);
        
        // 插入
        baseMapper.insert(user);
        
        // 根据ID查询
        User resultUser = baseMapper.selectById(1L);
        
        // 根据ID删除
        baseMapper.deleteById(1L);
        
        // 根据ID更新
        user.setId(1L);
        user.setName("李四");
        baseMapper.updateById(user);
    }
}
```

## 2、条件构造器（Wrapper）

这是 MP 的灵魂功能，用于动态构建 SQL 的 WHERE 条件，避免了手动拼接字符串的麻烦和错误。

**常用子类：**

- `QueryWrapper`：用于构建查询条件。
- `LambdaQueryWrapper`：使用 Lambda 表达式，防止字段名拼写错误（**推荐**）。

```java
// 使用 LambdaQueryWrapper (推荐，类型安全)
LambdaQueryWrapper<User> lqw = new LambdaQueryWrapper<>();
lqw
  .eq(User::getName, "张三")        // name = '张三'
  .gt(User::getAge, 20)           // AND age > 20
  .between(User::getCreateTime, startTime, endTime) // AND create_time BETWEEN ...
  .likeRight(User::getEmail, "zhang") // AND email LIKE 'zhang%'
  .orderByDesc(User::getCreateTime);  // ORDER BY create_time DESC

List<User> userList = userMapper.selectList(lqw);

// 使用 QueryWrapper
QueryWrapper<User> qw = new QueryWrapper<>();
qw.select("id", "name", "age") // 只查询特定字段
  .eq("name", "张三")
  .or(q -> q.gt("age", 20).isNotNull("email"));

List<User> userList = userMapper.selectList(qw);

// 使用 UpdateWrapper 进行更新
LambdaUpdateWrapper<User> luw = new LambdaUpdateWrapper<>();
luw
  .set(User::getEmail, "new_email@example.com") // SET email = ?
  .eq(User::getName, "张三");                   // WHERE name = ?
userMapper.update(null, luw); // 传入的entity为null，所有set由Wrapper指定
```

## 3、通用 Service

类似于通用 Mapper，你的 Service 接口可以继承 `IService`，实现类继承 `ServiceImpl`，获得更多批量操作的 Service 层方法。

```java
// 1. 定义Service接口
public interface UserService extends IService<User> {
}

// 2. 定义Service实现类
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
}

// 3. 使用
@Autowired
private UserService userService;

public void test() {
    // 批量插入
    List<User> userList = ...;
    userService.saveBatch(userList);
    
    // 链式查询
    List<User> list = userService.lambdaQuery()
        .eq(User::getStatus, 1)
        .list();
    
    // 链式更新
    boolean updated = userService.lambdaUpdate()
        .set(User::getEmail, "admin@example.com")
        .eq(User::getId, 1L)
        .update();
}
```

## 4、分页插件

内置了分页功能，需要先配置插件。

```java
// 1. 配置分页插件 (在配置类中)
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}

// 2. 使用分页
public Page<User> getUserByPage(int current, int size) {
    // current: 当前页码, size: 每页条数
    Page<User> page = new Page<>(current, size);
    
    LambdaQueryWrapper<User> lqw = new LambdaQueryWrapper<>();
    lqw.orderByDesc(User::getCreateTime);
    
    // 执行查询，page对象会被填充数据
    Page<User> resultPage = userMapper.selectPage(page, lqw);
    
    System.out.println("总记录数：" + resultPage.getTotal());
    System.out.println("总页数：" + resultPage.getPages());
    System.out.println("当前页数据：" + resultPage.getRecords());
    
    return resultPage;
}
```

依赖

```xml
3.5.9 PaginationInnerInterceptor 已分离出来。如需使用，则需单独引入 mybatis-plus-jsqlparser 依赖
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-jsqlparser-4.9</artifactId>
    <version>3.5.15</version>
</dependency>
```

## 5、ActiveRecord 模式

让实体类本身直接具备操作数据库的能力，适合简单的、单表操作较多的场景。

```java
// 1. 实体类继承 Model
@Data
@EqualsAndHashCode(callSuper = true)
public class User extends Model<User> {
    private Long id;
    private String name;
    private Integer age;
    
    // 2. 必须重写此方法，返回当前实体类的主键值
    @Override
    public Serializable pkVal() {
        return id;
    }
}

// 3. 直接通过实体类操作
User user = new User();
user.setName("王五");
user.insert(); // 插入

User resultUser = new User();
resultUser.setId(1L);
User userFromDB = resultUser.selectById(); // 查询

user.setAge(30);
user.updateById(); // 更新

user.deleteById(); // 删除
```

## 6、注解

用于配置实体类与数据库表的映射关系。

- `@TableName`：指定表名。
- `@TableId`：指定主键和主键策略。
- `@TableField`：指定字段映射。
- `@TableLogic`：逻辑删除注解。

```java
@TableName("sys_user") // 映射到表 sys_user
public class User {
    @TableId(value = "id", type = IdType.AUTO) // 主键，自增
    private Long id;
    
    @TableField("user_name") // 映射到字段 user_name
    private String name;
    
    private Integer age; // 默认驼峰转下划线 -> age
    
    @TableField(exist = false) // 表示该字段不是数据库的字段
    private String temporaryInfo;
    
    @TableLogic // 逻辑删除字段 (默认 0-未删除，1-已删除)
    private Integer deleted;
}
```

## 7、逻辑删除

```yml
# application.yml 配置
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted  # 全局逻辑删除的实体字段名
      logic-delete-value: 1        # 逻辑已删除值
      logic-not-delete-value: 0    # 逻辑未删除值
      
@TableLogic
private Integer deleted;
```

配置后，调用 `deleteById` 方法会自动变为 `UPDATE user SET deleted = 1 WHERE id = ?`。查询时会自动加上 `WHERE deleted = 0`。

## 8、自动填充

自动插入创建时间、更新时间等字段。

```java
// 1. 在实体类字段上添加注解
public class User {
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
    
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
}

// 2. 实现 MetaObjectHandler 接口
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

## 9、乐观锁

解决并发更新问题。

```java
// 1. 实体类中添加 @Version 注解
public class User {
    @Version
    private Integer version;
}

// 2. 配置插件
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor()); // 乐观锁插件
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL)); // 分页插件
    return interceptor;
}

// 3. 使用：先查询，再修改。MP会自动处理version
User user = userService.getById(1L);
user.setName("New Name");
boolean success = userService.updateById(user);
// UPDATE user SET name=?, version=? WHERE id=? AND version=? 此处如果更新失败 考虑报错或重试
```

## 10、动态数据源与多数据源事务

MyBatis-Plus 提供了 `dynamic-datasource-spring-boot-starter` 来简化多数据源配置。

**添加依赖**

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.5.2</version>
</dependency>
```

**数据源配置**

```yml
spring:
  datasource:
    dynamic:
      primary: master # 设置默认的数据源或者数据源组，默认值即为master
      strict: false # 严格模式，未匹配到数据源时抛异常，false时使用默认数据源
      datasource:
        master:
          url: jdbc:mysql://localhost:3306/master_db
          username: root
          password: 123456
          driver-class-name: com.mysql.cj.jdbc.Driver
        slave1:
          url: jdbc:mysql://localhost:3306/slave1_db
          username: root
          password: 123456
          driver-class-name: com.mysql.cj.jdbc.Driver
        slave2:
          url: jdbc:mysql://localhost:3306/slave2_db
          username: root
          password: 123456
          driver-class-name: com.mysql.cj.jdbc.Driver
        order_db:
          url: jdbc:mysql://localhost:3306/order_db
          username: root
          password: 123456
          driver-class-name: com.mysql.cj.jdbc.Driver
        user_db:
          url: jdbc:mysql://localhost:3306/user_db
          username: root
          password: 123456
          driver-class-name: com.mysql.cj.jdbc.Driver
```

**使用 `@DS` 注解切换数据源**

```java
@Service
public class UserService {
    
    // 默认使用 master 数据源
    public void defaultDataSource() {
        // 使用 master 数据源
    }
    
    @DS("slave1") // 指定使用 slave1 数据源
    public List<User> findFromSlave() {
        return userMapper.selectList(null);
    }
    
    @DS("user_db") // 使用 user_db 数据源
    public void userDbOperation() {
        // 在 user_db 中执行操作
    }
}

@DS("order_db") // 类级别注解，整个类都使用 order_db 数据源
@Service
public class OrderService {
    
    public void orderOperation() {
        // 使用 order_db 数据源
    }
    
    @DS("slave2") // 方法级别注解优先级更高，这个方法使用 slave2
    public List<Order> findOrdersFromSlave() {
        return orderMapper.selectList(null);
    }
}
```

**动态数据源事务管理**

使用 `@DSTransactional`

```java
@Service
public class MultiDataSourceService {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private OrderService orderService;
    
    /**
     * 多数据源分布式事务
     * 这个注解会使用分布式事务来管理多个数据源的操作
     */
    @DSTransactional
    public void multiDataSourceTransaction() {
        // 在 user_db 中操作
        userService.insertUser(new User(...));
        
        // 在 order_db 中操作  
        orderService.createOrder(new Order(...));
        
        // 如果这里抛出异常，两个数据源的操作都会回滚
        if (someCondition) {
            throw new RuntimeException("事务回滚");
        }
    }
    
    /**
     * 指定主数据源的多数据源事务
     */
    @DSTransactional
    public void multiDataSourceTxWithPrimary() {
        // 默认使用 master 数据源
        userService.defaultOperation();
        
        // 切换到 slave1
        userService.findFromSlave();
        
        // 所有操作都在同一个分布式事务中
    }
}
```

**编程式数据源切换**

```java
@Service
public class DynamicDataSourceService {
    
    @Autowired
    private DynamicDataSourceContextHolder dataSourceContextHolder;
    
    public void programmaticSwitch() {
        try {
            // 手动切换到指定数据源
            DynamicDataSourceContextHolder.push("slave1");
            
            // 执行操作，使用 slave1 数据源
            userService.someOperation();
            
        } finally {
            // 清理数据源，恢复默认
            DynamicDataSourceContextHolder.clear();
        }
    }
}
```

# 3、小技巧

## 1、update 字段为空方式

```java
1、配置文件
update-strategy: ignored

2、字段单独设置
@TableField(updateStrategy= FieldStrategy.IGNORED)

3、updateWrapper
LambdaUpdateWrapper<xxx> updateWrapper = new LambdaUpdateWrapper<>();
wrapper.eq(xxx::getId, xxx.getId());
wrapper.set(xxx::getBeginTime, null);
xxxMapper.update(xxx, updateWrapper);
```

# 4、扩展

## 1、拓展公共 Controller

## 2、拓展公共 Mapper 方法

**公共抽象方法**

```java
public abstract class BaseAbstractMethod extends AbstractMethod {
    public String prepareFieldSql(TableInfo tableInfo) {
        StringBuilder fieldSql = new StringBuilder();
        fieldSql.append(tableInfo.getKeyColumn()).append(",");
        tableInfo.getFieldList().forEach(x -> fieldSql.append(x.getColumn()).append(","));
        fieldSql.delete(fieldSql.length() - 1, fieldSql.length());
        fieldSql.insert(0, "(");
        fieldSql.append(")");
        return fieldSql.toString();
    }

    public String prepareValuesSqlForBatch(TableInfo tableInfo) {
        final StringBuilder valueSql = new StringBuilder();
        valueSql.append("<foreach collection=\"list\" item=\"item\" index=\"index\" open=\"(\" separator=\"),(\" close=\")\">");
        valueSql.append("#{item.").append(tableInfo.getKeyProperty()).append("},");
        tableInfo.getFieldList().forEach(x -> valueSql.append("#{item.").append(x.getProperty()).append("},"));
        valueSql.delete(valueSql.length() - 1, valueSql.length());
        valueSql.append("</foreach>");
        return valueSql.toString();
    }
}
```

**公共方法枚举**

```java
@Getter
public enum MySqlMethod {
    /**
     * 插入
     */
    SAVE_ALL_BATCH("saveAllBath", "批量插入", "<script>INSERT INTO %s %s VALUES %s</script>"),
    SAVE_ALL_BATCH_IGNORE("saveAllBathIgnore", "批量插入(ignore)", "<script>INSERT IGNORE INTO %s %s VALUES %s</script>"),
    SELECT_BY_ID_FOR_UPDATE("selectByIdForUpdate", "根据id 行锁查询", "SELECT %s FROM %s WHERE %s=#{%s} %s FOR UPDATE"),
    SELECT_BATCH_BY_IDS_FOR_UPDATE("selectBatchByIdsForUpdate", "根据id集合 行锁查询", "<script>SELECT %s FROM %s WHERE %s IN (%s) %s FOR UPDATE</script>");

    private final String method;
    private final String desc;
    private final String sql;

    MySqlMethod(String method, String desc, String sql) {
        this.method = method;
        this.desc = desc;
        this.sql = sql;
    }
}
```

**公共方法**

```java
public class SaveAllBath extends BaseAbstractMethod {
    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {
        final String sql = MySqlMethod.SAVE_ALL_BATCH.getSql();
        final String fieldSql = prepareFieldSql(tableInfo);
        final String valueSql = prepareValuesSqlForBatch(tableInfo);
        final String sqlResult = String.format(
                sql,
                tableInfo.getTableName(),
                fieldSql,
                valueSql
        );
        SqlSource sqlSource = languageDriver.createSqlSource(configuration, sqlResult, modelClass);
        return this.addInsertMappedStatement(mapperClass, modelClass, MySqlMethod.SAVE_ALL_BATCH.getMethod(), sqlSource, new NoKeyGenerator(), null, null);
    }
}

public class SaveAllBathIgnore extends BaseAbstractMethod {
    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {
        final String sql = MySqlMethod.SAVE_ALL_BATCH_IGNORE.getSql();
        final String fieldSql = prepareFieldSql(tableInfo);
        final String valueSql = prepareValuesSqlForBatch(tableInfo);
        final String sqlResult = String.format(
                sql,
                tableInfo.getTableName(),
                fieldSql,
                valueSql
        );
        SqlSource sqlSource = languageDriver.createSqlSource(configuration, sqlResult, modelClass);
        return this.addInsertMappedStatement(mapperClass, modelClass, MySqlMethod.SAVE_ALL_BATCH_IGNORE.getMethod(), sqlSource, new NoKeyGenerator(), null, null);
    }
}

public class SelectBatchByIdsForUpdate extends BaseAbstractMethod {
    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {
        MySqlMethod sqlMethod = MySqlMethod.SELECT_BATCH_BY_IDS_FOR_UPDATE;
        SqlSource sqlSource = languageDriver.createSqlSource(configuration,
                String.format(
                        sqlMethod.getSql(),
                        sqlSelectColumns(tableInfo, false),
                        tableInfo.getTableName(),
                        tableInfo.getKeyColumn(),
                        SqlScriptUtils.convertForeach("#{item}", "coll", null, "item", ","),
                        tableInfo.getLogicDeleteSql(true, true)
                ), Object.class);
        return addSelectMappedStatementForTable(mapperClass, sqlMethod.getMethod(), sqlSource, tableInfo);
    }
}

public class SelectByIdForUpdate extends BaseAbstractMethod {
    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {
        MySqlMethod sqlMethod = MySqlMethod.SELECT_BY_ID_FOR_UPDATE;
        SqlSource sqlSource = new RawSqlSource(configuration,
                String.format(
                        sqlMethod.getSql(),
                        sqlSelectColumns(tableInfo, false),
                        tableInfo.getTableName(),
                        tableInfo.getKeyColumn(), tableInfo.getKeyProperty(),
                        tableInfo.getLogicDeleteSql(true, true)
                ), Object.class);
        return this.addSelectMappedStatementForTable(mapperClass, sqlMethod.getMethod(), sqlSource, tableInfo);
    }
}
```

**公共方法注入**

```java
@Component
public class MySqlInjector extends DefaultSqlInjector {
    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass) {
        List<AbstractMethod> methodList = super.getMethodList(mapperClass);
        methodList.add(new SaveAllBath());
        methodList.add(new SaveAllBathIgnore());
        methodList.add(new SelectByIdForUpdate());
        methodList.add(new SelectBatchByIdsForUpdate());
        return methodList;
    }
}
```

**拓展 Mapper**

```java
public interface BasicMapper<T> extends BaseMapper<T> {
    /**
     * mysql批量插入
     *
     * @param list 集合
     * @return int
     */
    int saveAllBath(@Param("list") Collection<T> list);

    /**
     * mysql批量插入 ignore
     *
     * @param list 集合
     * @return int
     */
    int saveAllBathIgnore(@Param("list") Collection<T> list);

    /**
     * 根据id查询
     *
     * @param id id
     * @return {@code T}
     */
    T selectByIdForUpdate(Serializable id);

    /**
     * 根据id集合  行锁查询
     *
     * @param idList id集合
     * @return {@code List<T>}
     */
    List<T> selectBatchIdsForUpdate(@Param("coll") Collection<? extends Serializable> idList);
}
```

5、
