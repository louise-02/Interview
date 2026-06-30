# Redis 面试题

本文件可**单独阅读**，每题含完整参考答案。

深入学习 → [02-技术栈/Redis](../02-技术栈/Redis/Redis.md)；部署见 Deploy 相关文档。

---

### 1. Redis 单线程为什么快？

纯内存、IO 多路复用 epoll、高效数据结构、避免上下文切换（6.0+ 多线程仅 **IO** 非命令执行）。

---

### 2. 五种基本类型及场景？

String 缓存、计数；Hash 对象字段；List 队列、Timeline；Set 去重、交集；ZSet 排行榜、延迟队列。

---

### 3. 缓存穿透、击穿、雪崩？

**穿透**：查不存在数据，打穿到 DB → 布隆过滤器、空值缓存短 TTL。

**击穿**：热 key 过期瞬间高并发 → 互斥锁重建、逻辑不过期异步刷。

**雪崩**：大量 key 同时过期 → TTL 加随机、多级缓存、集群高可用。

---

### 4. RDB 和 AOF？

RDB 快照，恢复快，可能丢最后一次快照后数据。AOF 记命令，everysec 丢 1 秒，rewrite 压缩。**混合持久化** Redis4+ 推荐。

---

### 5. 缓存和数据库一致性？

**先更新 DB 再删缓存** 较常见；延迟双删；Canal 订阅 binlog 删缓存；强一致难，接受短暂不一致 + 过期 TTL。

---

### 6. 分布式锁 Redis 实现？

SET key uuid NX EX seconds 原子；解锁 Lua 比较 uuid 再 del，防删别人锁。Redisson 看门狗续期。主从切换极端情况锁丢失，需 Redlock 争议或 ZK。

---

### 7. Redis 过期策略和内存淘汰？

定期删除 + 惰性删除。内存达 maxmemory 按 policy：allkeys-lru、volatile-lru、allkeys-lfu 等。

---

### 8. 热 key 和大 key 问题？

热 key：本地缓存、多副本、拆分。大 key：拆 Hash、unlink 异步删、避免 hgetall 超大集合。

---

### 9. Redis 集群模式？

主从 + Sentinel 高可用；**Cluster** 16384 slot 分片，去中心化，客户端路由。跨 slot 多 key 事务受限。

---

### 10. 布隆过滤器原理？

多位数组 + 多 hash，判断**可能存在**或**一定不存在**，有误判无漏判，省内存防穿透。
