# Kafka 与消息队列面试题

本文件可**单独阅读**，每题含完整参考答案（Kafka 为主，含 RabbitMQ 等通用考点）。

深入学习 → [02-技术栈/Kafka](../02-技术栈/Kafka/1、Kafka%20简介.md)

---

### 1. 为什么用消息队列？

**异步**提速、**削峰**填谷、**解耦**上下游、最终一致。

---

### 2. 如何保证消息不丢？

生产：ack=all、重试；Broker：副本 ISR、min.insync.replicas；消费：**手动提交** offset，处理完再 commit。

---

### 3. 如何保证不重复消费？

消费端**幂等**：唯一键、Redis setnx、状态机。MQ 至少一次语义下重复不可避免，业务必须幂等。

---

### 4. 顺序消息怎么保证？

Kafka：**同一 key** 进同一分区，单分区单消费者线程。全局顺序则单分区（吞吐低）。

---

### 5. Kafka 为什么快？

顺序写磁盘、Page Cache、零拷贝 sendfile、分区并行、批量压缩。

---

### 6. RabbitMQ 和 Kafka 选型？

RabbitMQ：路由灵活、延迟插件、低延迟业务队列。Kafka：**高吞吐**日志流、大数据、事件溯源。订单通知 Rabbit；日志采集 Kafka。

---

### 7. 死信队列是什么？

消费失败 N 次或 TTL 过期进入 DLQ，人工排查或补偿，防 poison message 阻塞。

---

### 8. 延迟队列实现？

RabbitMQ TTL+DLX、Redis ZSet score=执行时间、Kafka 时间轮（较少原生延迟）。

---

### 9. 事务消息（RocketMQ 概念）？

半消息 + 本地事务 + commit/rollback，保证本地与发送一致。Kafka 事务 API 类似思想。

---

### 10. 消息积压怎么处理？

扩容消费者（≤分区数）、临时降级、跳过非核心、批量处理、查慢消费逻辑、加资源。
