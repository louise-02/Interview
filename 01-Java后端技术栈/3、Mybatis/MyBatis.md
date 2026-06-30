# 1、特性

## 1、拦截器

**完整的 SQL 执行流程及拦截点**

```
Executor → StatementHandler → ParameterHandler → ResultSetHandler

[Executor.update/query] 
    ↓
[StatementHandler.prepare] 
    ↓  
[StatementHandler.parameterize] 
    ↓
[ParameterHandler.setParameters] 
    ↓
[StatementHandler.update/query] 
    ↓
[ResultSetHandler.handleResultSets]
```

**Executor (执行器)**

作用：SQL 执行的顶层入口，负责整个 SQL 执行过程的调度，包括一二级缓存、事务管理等。

| 方法                 | 参数                                                         | 作用           | 拦截用途                      |
| :------------------- | :----------------------------------------------------------- | :------------- | :---------------------------- |
| `update`             | `MappedStatement ms, Object parameter`                       | 执行增删改操作 | 监控SQL性能、读写分离、多租户 |
| `query`              | `MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler` | 执行查询操作   | 分页插件、数据权限控制        |
| `query`              | `MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql` | 带缓存的查询   | 缓存管理、SQL重写             |
| `queryCursor`        | `MappedStatement ms, Object parameter, RowBounds rowBounds`  | 游标查询       | 大数据量查询优化              |
| `flushStatements`    | 无                                                           | 批量操作提交   | 批量操作监控                  |
| `commit`             | `boolean required`                                           | 提交事务       | 事务管理、日志记录            |
| `rollback`           | `boolean required`                                           | 回滚事务       | 事务回滚监控                  |
| `getTransaction`     | 无                                                           | 获取事务       | 事务状态监控                  |
| `close`              | `boolean forceRollback`                                      | 关闭执行器     | 资源清理监控                  |
| `isClosed`           | 无                                                           | 判断是否关闭   | 状态检查                      |
| `setExecutorWrapper` | `Executor executor`                                          | 设置包装器     | 执行器链管理                  |



**StatementHandler (语句处理器)**

作用：负责 SQL 语句的预处理、参数化设置和语句执行。

| 方法                  | 参数                                                | 作用            | 拦截用途          |
| :-------------------- | :-------------------------------------------------- | :-------------- | :---------------- |
| `prepare`             | `Connection connection, Integer transactionTimeout` | 创建Statement   | SQL改写、分页处理 |
| `parameterize`        | `Statement statement`                               | 参数化Statement | 参数预处理        |
| `batch`               | `Statement statement`                               | 批量执行        | 批量操作优化      |
| `update`              | `Statement statement`                               | 执行更新        | SQL监控、性能分析 |
| `query`               | `Statement statement, ResultHandler resultHandler`  | 执行查询        | 结果集预处理      |
| `queryCursor`         | `Statement statement`                               | 执行游标查询    | 游标控制          |
| `getBoundSql`         | 无                                                  | 获取BoundSql    | SQL获取和修改     |
| `getParameterHandler` | 无                                                  | 获取参数处理器  | 参数处理链        |



**ParameterHandler (参数处理器)**

作用：负责为 PreparedStatement 设置参数。

| 方法                 | 参数                   | 作用                        | 拦截用途         |
| :------------------- | :--------------------- | :-------------------------- | :--------------- |
| `getParameterObject` | 无                     | 获取参数对象                | 参数检查、修改   |
| `setParameters`      | `PreparedStatement ps` | 设置参数到PreparedStatement | 参数加密、格式化 |

**ResultSetHandler (结果集处理器)**

作用：负责将 ResultSet 结果集转换成 Java 对象。

| 方法                     | 参数                   | 作用                 | 拦截用途       |
| :----------------------- | :--------------------- | :------------------- | :------------- |
| `handleResultSets`       | `Statement stmt`       | 处理结果集           | 数据脱敏、解密 |
| `handleOutputParameters` | `CallableStatement cs` | 处理存储过程输出参数 | 输出参数处理   |
| `handleCursorResultSets` | `Statement stmt`       | 处理游标结果集       | 游标结果处理   |

**SQL 时间拦截器**

```java
@Slf4j
@Component
@Intercepts({
        @Signature(type = StatementHandler.class, method = "query", args = {Statement.class, ResultHandler.class}),
        @Signature(type = StatementHandler.class, method = "update", args = {Statement.class}),
        @Signature(type = StatementHandler.class, method = "batch", args = {Statement.class})
})
public class SqlTimeInterceptor implements Interceptor {
    private static final long SLOW_SQL_THRESHOLD = 1000L;
    private static final int MAX_SQL_LENGTH = 1000; // 限制SQL日志长度

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        BoundSql boundSql = statementHandler.getBoundSql();

        // 获取SQL和参数
        String originalSql = boundSql.getSql();
        Object parameterObject = boundSql.getParameterObject();
        String methodName = invocation.getMethod().getName();

        // 格式化SQL
        String formattedSql = formatSql(originalSql);

        long startTime = System.currentTimeMillis();

        try {
            Object result = invocation.proceed();
            // 成功执行，记录日志
            logSqlDetails(formattedSql, methodName, startTime, parameterObject, null);
            return result;
        } catch (Exception e) {
            // 执行异常，记录错误日志
            logSqlDetails(formattedSql, methodName, startTime, parameterObject, e);
            throw e;
        }
    }

    private String formatSql(String sql) {
        if (sql == null) return "";

        // 去除多余空格和换行
        String formatted = sql.replaceAll("\\s+", " ").trim();

        // 限制长度，避免日志过长
        if (formatted.length() > MAX_SQL_LENGTH) {
            formatted = formatted.substring(0, MAX_SQL_LENGTH) + "...";
        }

        return formatted;
    }

    private void logSqlDetails(String sql, String method, long startTime,
                               Object params, Exception error) {
        long costTime = System.currentTimeMillis() - startTime;

        // 构建日志消息
        if (error != null) {
            log.error("SQL执行异常 | 方法:{} | 耗时:{}ms | SQL:{} | 参数:{} | 错误:{}",
                    method, costTime, sql, params, error.getMessage());
        } else if (costTime > SLOW_SQL_THRESHOLD) {
            log.warn("慢SQL警告 | 方法:{} | 耗时:{}ms | SQL:{}",
                    method, costTime, sql);
        } else if (log.isDebugEnabled()) {
            log.debug("SQL执行 | 方法:{} | 耗时:{}ms | SQL:{}",
                    method, costTime, sql);
        }
    }
}
```



# 3、其他

## 1、Sql 解析

**依赖引入**

```xml
<dependency>
    <groupId>com.github.jsqlparser</groupId>
    <artifactId>jsqlparser</artifactId>
    <version>3.2</version>
</dependency>
```

**测试类**

```java
import net.sf.jsqlparser.JSQLParserException;
import net.sf.jsqlparser.expression.Alias;
import net.sf.jsqlparser.expression.Expression;
import net.sf.jsqlparser.parser.CCJSqlParserUtil;
import net.sf.jsqlparser.schema.Table;
import net.sf.jsqlparser.statement.Statement;
import net.sf.jsqlparser.statement.select.*;

import java.util.List;

public class SqlParserExample {
    public static void main(String[] args) throws JSQLParserException {
        String sql = "SELECT u.id, u.name, u.email, d.department_name " +
                "FROM users u " +
                "LEFT JOIN departments d ON u.department_id = d.id " +
                "WHERE u.status = 'ACTIVE' " +
                "  AND u.create_time >= '2024-01-01' " +
                "  AND (u.age > 18 OR u.role = 'ADMIN') " +
                "ORDER BY u.create_time DESC " + "LIMIT 10";

        // 使用工具类解析成Statement
        Statement statement = CCJSqlParserUtil.parse(sql);
        Select select = (Select) statement;

        // 获取body
        SelectBody body = select.getSelectBody();
        PlainSelect plainSelect = (PlainSelect) body;

        // 获取From
        FromItem fromItem = plainSelect.getFromItem();

        // 获取表名
        Table table;
        if (fromItem instanceof Table) {
            table = (Table) fromItem;
            String tableName = table.getName();
            System.out.println("主表名: " + tableName); // 输出: users
        }

        // 获取别名
        Alias alias = fromItem.getAlias();
        if (alias != null) {
            System.out.println("主表别名: " + alias.getName()); // 输出: u
        }

        // 获取Join信息
        List<Join> joins = plainSelect.getJoins();
        if (joins != null) {
            for (Join join : joins) {
                FromItem joinItem = join.getRightItem();
                if (joinItem instanceof Table) {
                    Table joinTable = (Table) joinItem;
                    System.out.println("关联表: " + joinTable.getName() + " 别名: " + joinTable.getAlias().getName());
                    // 输出: 关联表: departments 别名: d
                }
            }
        }

        // 获取where条件
        Expression where = plainSelect.getWhere();
        System.out.println("原始WHERE条件: " + where);
        // 输出: u.status = 'ACTIVE' AND u.create_time >= '2024-01-01' AND (u.age > 18 OR u.role = 'ADMIN')

        // 重新拼接where条件
        StringBuilder sqlBuilder = new StringBuilder();
        if (where == null) {
            sqlBuilder.append(" 1=1");
        } else {
            sqlBuilder.append(where);
        }

        // 进行where条件处理 - 添加额外的过滤条件
        System.out.println("处理前WHERE: " + sqlBuilder);

        // 示例1: 添加数据权限过滤
        sqlBuilder.append(" AND u.org_id = 1001");

        // 示例2: 添加多租户过滤
        sqlBuilder.append(" AND u.tenant_id = 'company_a'");

        // 示例3: 添加逻辑删除过滤
        sqlBuilder.append(" AND u.deleted = 0");

        System.out.println("处理后WHERE: " + sqlBuilder);

        // 重新生成where条件
        Expression expression = CCJSqlParserUtil.parseCondExpression(sqlBuilder.toString());
        plainSelect.setWhere(expression);

        // 输出最终SQL
        System.out.println("最终SQL: " + select);
    }
}
```

