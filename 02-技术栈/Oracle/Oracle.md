# 1、表结构同步

**1、查看目录权限**

查看所有目录对象（DBA权限）

```sql
-- 查看所有目录对象
SELECT * FROM DBA_DIRECTORIES;

-- 查看目录权限分配
SELECT * FROM DBA_TAB_PRIVS 
WHERE TABLE_NAME IN (SELECT DIRECTORY_NAME FROM DBA_DIRECTORIES);
```

查看当前用户可用的目录

```sql
-- 查看当前用户有权限的目录
SELECT * FROM ALL_DIRECTORIES;

-- 查看当前用户对具体目录的权限
SELECT * FROM USER_TAB_PRIVS 
WHERE TABLE_NAME = 'DATA_PUMP_DIR';
```

查看当前用户权限

```sql
-- 查看当前用户角色
SELECT * FROM USER_ROLE_PRIVS;

-- 查看系统权限
SELECT * FROM USER_SYS_PRIVS;
```

**2、目录授权管理**

创建目录对象（需要DBA权限）

```sql
-- 创建目录
CREATE OR REPLACE DIRECTORY DATA_PUMP_DIR AS '/u01/app/oracle/dpdump/';

-- 创建用户专属目录
CREATE OR REPLACE DIRECTORY USER_DUMP_DIR AS '/home/oracle/user_dump/';
```

授权给用户（需要DBA权限）

```sql
-- 授权单个目录给用户
GRANT READ, WRITE ON DIRECTORY DATA_PUMP_DIR TO VLMP_TEST;
GRANT READ, WRITE ON DIRECTORY DATA_PUMP_DIR TO VLMP;

-- 授权给多个用户
GRANT READ, WRITE ON DIRECTORY DATA_PUMP_DIR TO VLMP_TEST, VLMP;

-- 授权CREATE ANY DIRECTORY权限（谨慎）
GRANT CREATE ANY DIRECTORY TO VLMP_TEST;
```

撤销权限

```sql
-- 撤销目录权限
REVOKE READ, WRITE ON DIRECTORY DATA_PUMP_DIR FROM VLMP_TEST;

-- 删除目录
DROP DIRECTORY DATA_PUMP_DIR;
```

**3、数据导出导入操作**

多个表

```sql
# 基本语法
expdp 用户名/密码@数据库连接串 \
DIRECTORY=目录名 \
DUMPFILE=导出文件名.dmp \
LOGFILE=日志文件名.log \
TABLES=表1,表2,表3

# 实际例子
expdp VLMP_TEST/VLMP_Aa123456@10.91.11.106:1521/ORCLCDB \
DIRECTORY=DATA_PUMP_DIR \
DUMPFILE=multiple_tables.dmp \
LOGFILE=export_tables.log \
TABLES=BASE_ASSESSMENT_CONFIG,BASE_ASSESSMENT_DEDUCTION_RULE,BASE_ASSESSMENT_PROJECT \
CONTENT=ALL

# 导入指定表
impdp VLMP/VLMP_Aa123456@10.1.200.188:1521/ORCLCDB \
DIRECTORY=DATA_PUMP_DIR \
DUMPFILE=multiple_tables.dmp \
LOGFILE=import_tables.log \
TABLES=BASE_ASSESSMENT_CONFIG,BASE_ASSESSMENT_DEDUCTION_RULE,BASE_ASSESSMENT_PROJECT \
REMAP_SCHEMA=VLMP_TEST:VLMP \
TABLE_EXISTS_ACTION=REPLACE \
TRANSFORM=SEGMENT_ATTRIBUTES:N

# 导入指定表 SCHEMA 转换
impdp VLMP/VLMP_Aa123456@10.1.200.188:1521/ORCLCDB \
DIRECTORY=DATA_PUMP_DIR \
DUMPFILE=multiple_tables.dmp \
LOGFILE=import_tables.log \
TABLES=VLMP_TEST.BASE_ASSESSMENT_CONFIG,VLMP_TEST.BASE_ASSESSMENT_DEDUCTION_RULE,VLMP_TEST.BASE_ASSESSMENT_PROJECT \
REMAP_SCHEMA=VLMP_TEST:VLMP \
TABLE_EXISTS_ACTION=REPLACE \
TRANSFORM=SEGMENT_ATTRIBUTES:N
```

整个schema

```sql
# 导出整个用户下的所有对象
expdp VLMP_TEST/VLMP_Aa123456@10.91.11.106:1521/ORCLCDB \
DIRECTORY=DATA_PUMP_DIR \
DUMPFILE=full_schema.dmp \
LOGFILE=export_schema.log \
SCHEMAS=VLMP_TEST \
CONTENT=ALL

# 导入整个schema
impdp VLMP/VLMP_Aa123456@10.1.200.188:1521/ORCLCDB \
DIRECTORY=DATA_PUMP_DIR \
DUMPFILE=full_schema.dmp \
LOGFILE=import_schema.log \
SCHEMAS=VLMP_TEST \
REMAP_SCHEMA=VLMP_TEST:VLMP \
TABLE_EXISTS_ACTION=REPLACE \
TRANSFORM=SEGMENT_ATTRIBUTES:N
```

整个数据库

```sql
# 需要DBA权限
expdp system/密码@10.91.11.106:1521/ORCLCDB \
DIRECTORY=DATA_PUMP_DIR \
DUMPFILE=full_database.dmp \
LOGFILE=export_full.log \
FULL=Y \
CONTENT=ALL

# 需要DBA权限
impdp system/密码@10.1.200.188:1521/ORCLCDB \
DIRECTORY=DATA_PUMP_DIR \
DUMPFILE=full_database.dmp \
LOGFILE=import_full.log \
FULL=Y \
TABLE_EXISTS_ACTION=REPLACE \
TRANSFORM=SEGMENT_ATTRIBUTES:N
```

导出参数

| 参数          | 说明             | 示例                                                         |
| :------------ | :--------------- | :----------------------------------------------------------- |
| `DIRECTORY`   | 目录对象名       | `DATA_PUMP_DIR`                                              |
| `DUMPFILE`    | 导出文件名       | `export.dmp`                                                 |
| `LOGFILE`     | 日志文件名       | `export.log`                                                 |
| `TABLES`      | 导出的表列表     | `table1,table2`                                              |
| `SCHEMAS`     | 导出的schema列表 | `user1,user2`                                                |
| `FULL`        | 导出整个数据库   | `Y`                                                          |
| `CONTENT`     | 导出内容         | `ALL`(数据+结构) `DATA_ONLY`(仅数据) `METADATA_ONLY`(仅结构) |
| `QUERY`       | 条件导出         | `"WHERE date_col > '2023-01-01'"`                            |
| `EXCLUDE`     | 排除对象         | `TABLE:"IN ('TEMP%')"`                                       |
| `INCLUDE`     | 包含对象         | `TABLE:"LIKE 'IMPORT%'"`                                     |
| `PARALLEL`    | 并行度           | `4`                                                          |
| `COMPRESSION` | 压缩             | `ALL`                                                        |

导入参数

| 参数                  | 说明           | 示例                                                         |
| :-------------------- | :------------- | :----------------------------------------------------------- |
| `TABLE_EXISTS_ACTION` | 表存在时的动作 | `SKIP`(跳过) `APPEND`(追加数据) `TRUNCATE`(清空后插入) `REPLACE`(删除重建) |
| `REMAP_SCHEMA`        | schema映射     | `source_user:target_user`                                    |
| `REMAP_TABLESPACE`    | 表空间映射     | `source_ts:target_ts`                                        |
| `TRANSFORM`           | 对象转换       | `SEGMENT_ATTRIBUTES:N`(移除存储属性)                         |
| `REMAP_TABLE`         | 表名映射       | `old_name:new_name`                                          |