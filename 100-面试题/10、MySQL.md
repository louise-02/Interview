# MySQL 面试题

本文件可**单独阅读**，每题含完整参考答案。

深入学习 → [02-技术栈/Mysql](../02-技术栈/Mysql/Mysql基础.md)（知识体系重构中）；部署 → [Deploy/3、mysql.md](../05-运维和部署/Deploy/3、mysql.md)

---

### 1. InnoDB 和 MyISAM 区别？

InnoDB：**行锁**、事务、外键、聚簇索引、MVCC，生产默认。MyISAM：表锁、无事务、非聚簇，读多写少老场景，现已少用。

---

### 2. 聚簇索引和非聚簇索引？

InnoDB **主键索引叶子存整行数据**（聚簇）。二级索引叶子存**主键值**，查非覆盖索引需**回表**再查聚簇。主键宜短、自增，避免 UUID 随机插入页分裂。

---

### 3. 什么是覆盖索引？

查询列全在索引树中，不需回表，Extra 显示 Using index。如索引 (user_id, status) 查 select user_id, status where user_id=?。

---

### 4. 索引失效常见情况？

对列**函数/运算**、隐式类型转换（字符串列不加引号）、like **左模糊** %xx、or 一侧无索引、违反最左前缀（联合索引跳过左侧列）、优化器判全表扫描更优。

---

### 5. 事务 ACID？

原子性 undo log、一致性业务+约束、隔离性 MVCC/锁、持久性 redo log。

---

### 6. 隔离级别与现象？

RU 脏读；RC 不可重复读；**RR** 默认 InnoDB 防幻读（快照读+间隙锁当前读）；Serializable 串行。RC 锁更少，RR 默认。

---

### 7. MVCC 原理？

隐藏列 trx_id、roll_pointer，undo 版本链。**快照读** ReadView 判断可见性；**当前读** select for update 加锁读最新。

---

### 8. 慢 SQL 怎么优化？

explain 看 type、key、rows、Extra；加/改索引、避免 select *、拆分大 SQL、读写分离、归档历史数据。

---

### 9. 主从复制原理？

binlog → 从库 IO 线程拉 relay log → SQL 线程重放。异步默认；半同步至少一个从库 ack。延迟监控 Seconds_Behind_Master。

---

### 10. 分库分表思路？

垂直拆库按业务；水平拆表按 user_id 取模/范围。中间件 ShardingSphere。分布式 ID、跨库 join 难、全局事务复杂。

---

### 11. EXPLAIN 中 type 哪些好？

system > const > eq_ref > ref > range > index > ALL，尽量 ref 以上，避免 ALL 大表。

---

### 12. redo log 和 binlog 区别？

redo InnoDB 物理页崩溃恢复；binlog Server 层逻辑复制、审计。两阶段提交保证一致。

---

### 13. 什么是间隙锁？

RR 下当前读防幻读，锁住索引间隙。可能死锁，RC 无间隙锁。

---

### 14. 数据库连接池为什么需要？

建连昂贵，池化复用 HikariCP/Druid，控制最大连接防打满 DB。

---

### 15. 读写分离怎么实现？

主写从读，中间件或 @Transactional(readOnly=true) 路由，注意**主从延迟**读不到刚写数据 → 强制走主或缓存。
