# 1、添加依赖

Spring Boot 项目引入 `spring-kafka` 即可（版本由 Boot 统一管理）：

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

启用监听需要加 `@EnableKafka`（可放在启动类或任意 `@Configuration` 上）：

```java
@SpringBootApplication
@EnableKafka
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

# 2、yml 配置（推荐模板）

第 2 节是**手动提交 + 批量消费**的推荐基线；第 3 节起为不同场景的独立示例，**不要全部 yml 叠在一起用**。

> 配置分三层：**Producer/Consumer** 在本 yml；**Broker** 在 `server.properties` 和建 topic 时；防丢防重总览见 **6、丢消息与重复消费.md**。

```yml
spring:
  kafka:
    # 【公共】Kafka Broker 地址，多个用逗号分隔；Producer/Consumer 都用它发现集群
    bootstrap-servers: 127.0.0.1:9092
    producer:
      # 【生产者】序列化 key 的类（一般 String 或 Long）
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      # 【生产者】序列化 value 的类
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      # 【生产者】消息确认机制
      # 0：不等待任何确认，性能高但可能丢数据
      # 1：等待 Leader 副本确认，性能与安全性折中
      # all / -1：等待 ISR 全部副本确认，最安全但性能最低（防丢推荐）
      acks: all
      # 【生产者】发送失败时自动重试次数（开启幂等后可能被 broker 侧策略覆盖）
      retries: 3
      # 【生产者】单次 Produce 请求最大字节，须 ≤ Broker message.max.bytes
      max-request-size: 1048576
      properties:
        # 【生产者】开启幂等性，避免同分区重复 record；会强制 acks=all
        enable.idempotence: true
        # 【生产者】单连接上未收到 ack 的 in-flight 请求数；幂等时 1~5；严格有序可设 1
        max.in.flight.requests.per.connection: 5
        # 【生产者】消息发送前等待时间（ms），与 batch-size 配合形成批量发送
        linger.ms: 20
        # 【生产者】单次请求等待 Broker 响应的最大时间（ms）
        request.timeout.ms: 60000
        # 【生产者】从 send 到最终成功/失败的总超时（ms），包含 retry
        delivery.timeout.ms: 120000
        # 【生产者】每次发送失败后等待多长时间再重试（ms）
        retry.backoff.ms: 1000
        # 【生产者】获取 metadata 的最大阻塞时间（ms）；topic 不存在、集群不可达时会阻塞到此上限
        max.block.ms: 20000
        # 【生产者】压缩算法：none、gzip、snappy、lz4、zstd
        compression.type: lz4
      # 【生产者】批量发送时单个 batch 的最大字节数（默认 16384）
      batch-size: 65536
      # 【生产者】缓冲区大小，缓存尚未发送的消息（默认 33554432 = 32MB）
      buffer-memory: 67108864
    consumer:
      # 【消费者】所属消费组 ID；Kafka 根据 group.id 管理 offset 和 rebalance
      group-id: my-group
      # 【消费者】当没有 offset 时的消费策略（首次启动或 offset 丢失时）
      # earliest：从分区最早记录开始消费
      # latest：从最新记录开始，只消费启动后产生的数据
      # none：没有有效 offset 时抛异常，不消费任何数据
      auto-offset-reset: earliest
      # 【消费者】是否自动提交 offset
      # true：后台定时提交，简单但可能丢/重复
      # false：手动 ack，控制力更强（推荐）
      enable-auto-commit: false
      # 【消费者】反序列化 key 的类
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      # 【消费者】反序列化 value 的类
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      # 【消费者】一次 poll 最多拉取的消息数（默认 500）；batch 消费 + 处理慢时别过大
      max-poll-records: 500
      # 【消费者】等待拉取时间上限（ms）；与 fetch-min-size 配合，默认 500
      # 若 Kafka 没有足够消息，则等待此时间以形成批量
      fetch-max-wait: 500
      # 【消费者】单次拉取的最小字节数，达到此值才返回；低流量 topic 过大可能增加延迟
      fetch-min-size: 1048576
      properties:
        # 【消费者】每个分区单次最多拉取的字节数，限制返回数据大小，避免 OOM
        max.partition.fetch.bytes: 10485760
        # 【消费者】两次 poll 之间业务处理最长时间（ms），默认 5 分钟；超时踢出组 rebalance
        max.poll.interval.ms: 300000
        # 【消费者】与 coordinator 连接超时（ms），默认 45s；超时 consumer 被移除并 rebalance
        session.timeout.ms: 45000
        # 【消费者】心跳间隔（ms），默认 3s；须 < session.timeout.ms，建议 ≤ 其 1/3
        heartbeat.interval.ms: 15000
        # 【消费者】事务隔离级别
        # read_uncommitted：可读到未提交事务的消息（默认）
        # read_committed：只读已提交事务的消息（消费事务 Producer 写入的 topic 时用）
        isolation.level: read_uncommitted
        # 【消费者】订阅不存在的 topic 时是否自动创建；生产建议 false
        allow.auto.create.topics: false
    listener:
      # 【Listener】监听模式
      # single：单条消费
      # batch：批量消费，配合 max-poll-records / fetch-max-wait / fetch-min-size
      type: batch
      # 【Listener】ack 模式（常配合 enable-auto-commit=false）
      # manual_immediate：手动 ack，立即提交 offset
      # manual：手动 ack，延迟到下一次 poll 前提交
      # record：每处理完一条 ack 一次
      # batch：处理完一批 ack 一次
      # time / count / count_time：定时或条数触发 ack
      ack-mode: manual_immediate
      # 【Listener】并发线程数，不超过 topic 分区数
      # concurrency: 3
```

**本 yml 未包含、按需添加**

| 配置 | 何时加 |
|------|--------|
| `producer.transaction-id-prefix` | Kafka 事务发送 |
| `spring.kafka.security.*` | SASL / SSL 认证 |
| `client-id` | 多应用共集群时区分客户端 |

---



# 3、自动提交 消费者

## 单条

```yml
spring:
  kafka:
    consumer:
      enable-auto-commit: true
      auto-offset-reset: earliest
      group-id: group1
```

```java
@Component
public class SingleAutoAckListener {

    @KafkaListener(topics = "topic1", groupId = "group1")
    public void listen(String message) {
        System.out.println("收到消息：" + message);
        // 自动提交 offset，无需手动 ack
    }
}
```

## 批量

```yml
spring:
  kafka:
    consumer:
      enable-auto-commit: true
      auto-offset-reset: earliest
      group-id: group1
    listener:
      type: batch
```

```java
@Component
public class BatchAutoAckListener {

    @KafkaListener(topics = "topic1", groupId = "group1")
    public void listen(List<String> messages) {
        System.out.println("收到批量消息，数量：" + messages.size());
        messages.forEach(System.out::println);
    }
}
```

---

# 4、手动提交 消费者

## 单条

```yml
spring:
  kafka:
    consumer:
      enable-auto-commit: false
      auto-offset-reset: earliest
      group-id: group1
    listener:
      ack-mode: manual_immediate
```

```java
@Component
public class SingleManualAckListener {

    @KafkaListener(topics = "topic1", groupId = "group1")
    public void listen(ConsumerRecord<String, String> record, Acknowledgment ack) {
        System.out.println("收到消息：" + record.value());
        ack.acknowledge();
    }
}
```

## 批量

```yml
spring:
  kafka:
    consumer:
      enable-auto-commit: false
      auto-offset-reset: earliest
      group-id: group1
    listener:
      type: batch
      ack-mode: manual_immediate
```

```java
@Component
public class BatchManualAckListener {

    @KafkaListener(topics = "topic1", groupId = "group1")
    public void listen(List<ConsumerRecord<String, String>> records, Acknowledgment ack) {
        System.out.println("收到批量消息，数量：" + records.size());
        for (ConsumerRecord<String, String> record : records) {
            System.out.println(record.value());
        }
        ack.acknowledge();
    }
}
```

---

# 5、生产者

## 单条发送（异步回调）

```java
@Service
public class KafkaProducerService {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessageWithCallback(String topic, String key, String value) {
        kafkaTemplate.send(topic, key, value).whenComplete((result, ex) -> {
            if (ex != null) {
                System.err.println("消息发送失败：" + ex.getMessage());
                return;
            }
            System.out.println("消息发送成功，offset=" + result.getRecordMetadata().offset());
        });
    }
}
```

> 旧版可用 `addCallback`；Spring Kafka 2.8+ 推荐 `whenComplete`。

## 循环多条发送（非事务，依赖 Producer 底层 batching）

本质是多次 `send()`，由 `linger.ms` / `batch-size` 在客户端合并，**不是 Kafka 事务意义上的「一批」**。

```java
@Service
public class KafkaMultiSendProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMany(String topic, Map<String, String> messages) {
        for (Map.Entry<String, String> entry : messages.entrySet()) {
            kafkaTemplate.send(topic, entry.getKey(), entry.getValue())
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        System.err.println("发送失败：" + ex.getMessage());
                    }
                });
        }
        // 需要确保全部刷出缓冲区时可 kafkaTemplate.flush()
    }
}
```

## 事务发送（要么全成功，要么全失败）

```yml
spring:
  kafka:
    producer:
      acks: all
      transaction-id-prefix: tx-
      properties:
        enable.idempotence: true
```

```java
@Service
public class KafkaTransactionalProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendBatchWithTransaction(String topic, Map<String, String> messages) {
        kafkaTemplate.executeInTransaction(operations -> {
            for (Map.Entry<String, String> entry : messages.entrySet()) {
                // 事务内不要用异步 callback；异常会直接回滚整笔事务
                operations.send(topic, entry.getKey(), entry.getValue());
            }
            return true;
        });
    }
}
```

---

# 6、消费者异常处理 / 死信队列

## 配置类

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.listener.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.listener.ContainerProperties;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.util.backoff.FixedBackOff;

@Configuration
public class KafkaListenerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> batchKafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory,
            KafkaTemplate<String, String> kafkaTemplate) {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
                new DeadLetterPublishingRecoverer(kafkaTemplate),
                new FixedBackOff(1000L, 3L)
        );
        errorHandler.setCommitRecovered(true);
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> singleKafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory,
            KafkaTemplate<String, String> kafkaTemplate) {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(false);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
                new DeadLetterPublishingRecoverer(kafkaTemplate),
                new FixedBackOff(1000L, 3L)
        );
        errorHandler.setCommitRecovered(true);
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }
}
```

说明：

- `DeadLetterPublishingRecoverer`：重试耗尽后转发到死信 topic，默认命名一般为 **`原topic名.DLT`**（需提前创建或配置自动建 topic）。
- `FixedBackOff(1000L, 3L)`：间隔 1 秒，最多重试 **3 次**（不含首次消费）。
- `setCommitRecovered(true)`：进 DLQ 成功后提交 offset，避免同一条反复消费。
- **批量消费**下，一条失败可能导致**整批**重试/进 DLQ，和单条语义不同。

## 使用

`enable-auto-commit=false` 且工厂为 `MANUAL_IMMEDIATE` 时，listener **必须**传 `Acknowledgment` 并调用 `ack.acknowledge()`。

```java
@Component
public class MyKafkaListeners {

    @KafkaListener(topics = "my-topic", containerFactory = "batchKafkaListenerContainerFactory")
    public void listenBatch(List<ConsumerRecord<String, String>> records, Acknowledgment ack) {
        // 批量处理；任一条抛异常会触发 DefaultErrorHandler
        ack.acknowledge();
    }

    @KafkaListener(topics = "my-topic", containerFactory = "singleKafkaListenerContainerFactory")
    public void listenSingle(ConsumerRecord<String, String> record, Acknowledgment ack) {
        ack.acknowledge();
    }
}
```

## Spring Kafka 2.7 及以前（旧 API）

指的是 **Spring Kafka** 版本，不是 Kafka Broker 2.8。  
Spring Kafka **2.8+** 请用上一节的 `DefaultErrorHandler` + `setCommonErrorHandler`。

旧版 API 对照：

| 2.7 及以前 | 2.8+ |
|------------|------|
| `setErrorHandler` | `setCommonErrorHandler` |
| `setBatchErrorHandler` | `setCommonErrorHandler`（同一 handler） |
| `SeekToCurrentErrorHandler` | `DefaultErrorHandler` |
| `RecoveringBatchErrorHandler` | `DefaultErrorHandler` |
| `setAckAfterHandle(true)` | `setCommitRecovered(true)` |

### yml（与新版相同）

```yml
spring:
  kafka:
    consumer:
      enable-auto-commit: false
      group-id: my-group
    listener:
      ack-mode: manual_immediate
```

### 单条消费 + DLQ（SeekToCurrentErrorHandler）

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.listener.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.listener.ContainerProperties;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.listener.SeekToCurrentErrorHandler;
import org.springframework.util.backoff.FixedBackOff;

@Configuration
public class KafkaLegacyListenerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> singleKafkaListenerContainerFactoryLegacy(
            ConsumerFactory<String, String> consumerFactory,
            KafkaTemplate<String, String> kafkaTemplate) {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(false);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
        SeekToCurrentErrorHandler errorHandler = new SeekToCurrentErrorHandler(
                recoverer,
                new FixedBackOff(1000L, 3L)
        );
        // 进 DLQ 成功后提交 offset，等价于 DefaultErrorHandler#setCommitRecovered(true)
        errorHandler.setAckAfterHandle(true);

        factory.setErrorHandler(errorHandler);
        return factory;
    }
}
```

### 批量消费 + DLQ（RecoveringBatchErrorHandler）

批量 listener 不要用 `SeekToCurrentErrorHandler`，旧版应使用 `RecoveringBatchErrorHandler`：

```java
import org.springframework.kafka.listener.RecoveringBatchErrorHandler;

@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> batchKafkaListenerContainerFactoryLegacy(
        ConsumerFactory<String, String> consumerFactory,
        KafkaTemplate<String, String> kafkaTemplate) {

    ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    factory.setBatchListener(true);
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
    RecoveringBatchErrorHandler errorHandler = new RecoveringBatchErrorHandler(
            recoverer,
            new FixedBackOff(1000L, 3L)
    );

    factory.setBatchErrorHandler(errorHandler);
    return factory;
}
```

### 使用

```java
@Component
public class MyLegacyKafkaListeners {

    @KafkaListener(topics = "my-topic", containerFactory = "singleKafkaListenerContainerFactoryLegacy")
    public void listenSingle(ConsumerRecord<String, String> record, Acknowledgment ack) {
        // 业务处理；失败则重试 3 次后进 my-topic.DLT
        ack.acknowledge();
    }

    @KafkaListener(topics = "my-topic", containerFactory = "batchKafkaListenerContainerFactoryLegacy")
    public void listenBatch(List<ConsumerRecord<String, String>> records, Acknowledgment ack) {
        ack.acknowledge();
    }
}
```

### 自定义 DLQ topic（可选）

默认发到 `原topic.DLT`；可自定义目标分区：

```java
import org.apache.kafka.common.TopicPartition;

DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
        kafkaTemplate,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
);
SeekToCurrentErrorHandler errorHandler = new SeekToCurrentErrorHandler(
        recoverer,
        new FixedBackOff(1000L, 3L)
);
errorHandler.setAckAfterHandle(true);
```

---

# 7、动态控制 Kafka 消费启停（配置中心 / Redis）

通过外部配置（如 Redis、Nacos）开关消费。示例里 `CommonConfigExpander.getFtVehicleDataEnable()` 表示从配置中心读取开关，**请按项目实际替换**。

## 消费者

`id` 是监听器唯一标识，**不要**与 `groupId` 混用；`getListenerContainer(id)` 按 `id` 查找。

```java
public static final String FT_VEHICLE_DATA_LISTENER_ID = "ftVehicleDataListener";

@KafkaListener(
        id = FT_VEHICLE_DATA_LISTENER_ID,
        groupId = "${spring.kafka.consumer.group-id}",
        topics = "ft_vehicle_data",
        containerFactory = "singleKafkaListenerContainerFactory",
        autoStartup = "false"
)
public void listen(ConsumerRecord<String, String> record, Acknowledgment ack) {
    // 业务处理
    ack.acknowledge();
}
```

## 启停调度

- `start()` / `stop()`：启动或停止容器（停止会释放分区）。
- `pause()` / `resume()`：暂停/恢复拉取，**仍占用分区**，适合临时停消费。
- 需要 `@EnableScheduling` 才会执行定时检查。

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.kafka.config.KafkaListenerEndpointRegistry;
import org.springframework.kafka.listener.MessageListenerContainer;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@EnableScheduling
public class KafkaConsumerPauseResumeScheduler {

    private static final String LISTENER_ID = "ftVehicleDataListener";

    private final KafkaListenerEndpointRegistry registry;
    private Boolean lastState = null;

    public KafkaConsumerPauseResumeScheduler(KafkaListenerEndpointRegistry registry) {
        this.registry = registry;
    }

    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        applyState(CommonConfigExpander.getFtVehicleDataEnable(), "[Init]");
    }

    @Scheduled(fixedDelay = 5000)
    public void checkAndControlConsumer() {
        boolean enabled = CommonConfigExpander.getFtVehicleDataEnable();
        if (lastState != null && lastState == enabled) {
            return;
        }
        applyState(enabled, "[Scheduler]");
        lastState = enabled;
    }

    private void applyState(boolean enabled, String tag) {
        MessageListenerContainer container = registry.getListenerContainer(LISTENER_ID);
        if (container == null) {
            log.warn("{} listener not found: {}", tag, LISTENER_ID);
            return;
        }
        if (enabled) {
            if (!container.isRunning()) {
                container.start();
                log.info("{} started {}", tag, LISTENER_ID);
            } else if (container.isPauseRequested()) {
                container.resume();
                log.info("{} resumed {}", tag, LISTENER_ID);
            }
        } else if (container.isRunning()) {
            // 临时停消费用 pause；若要释放分区可改为 container.stop()
            container.pause();
            log.info("{} paused {}", tag, LISTENER_ID);
        }
    }
}
```

> `registry.getListenerContainer(...)` 的参数是 `@KafkaListener` 的 **`id`**，不是 `groupId`。

---

# 8、防丢消息与重复消费

Spring 侧 yml 只覆盖 Producer/Consumer；Broker 副本、`min.insync.replicas` 等在 `server.properties` 配置。  
完整三层对照、参数说明、业务幂等落地见 **6、丢消息与重复消费.md**。
