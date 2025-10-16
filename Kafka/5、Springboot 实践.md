# 说明

使用 Redis 中标识，来控制 Kafka 的消费启停。

# 配置文件

消费者

```java
@KafkaListener(
        id = "${spring.kafka.consumer.group-id}",
        groupId = "${spring.kafka.consumer.group-id}",
        topics = "ft_vehicle_data",
        containerFactory = "kafkaListenerContainerFactory",
        autoStartup = "false"
)
```

启停配置

```java
import com.baiccl.cermp.business.common.config.CommonConfigExpander;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.kafka.config.KafkaListenerEndpointRegistry;
import org.springframework.kafka.listener.MessageListenerContainer;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

/**
 * 控制kafka消费
 *
 * @author louise
 * @since 2025/5/19
 */
@Slf4j
@Component
public class KafkaConsumerPauseResumeScheduler implements ApplicationListener<ContextRefreshedEvent> {
    @Value("${spring.kafka.consumer.group-id}")
    private String kafkaGroupId;

    private final KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

    private Boolean lastState = null;

    public KafkaConsumerPauseResumeScheduler(KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry) {
        this.kafkaListenerEndpointRegistry = kafkaListenerEndpointRegistry;
    }

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // Spring 上下文加载完成时执行
        boolean enabled = CommonConfigExpander.getFtVehicleDataEnable();

        MessageListenerContainer container = kafkaListenerEndpointRegistry.getListenerContainer(kafkaGroupId);
        if (container == null) {
            log.info("[Init] no consumer {}", kafkaGroupId);
            return;
        }

        if (enabled) {
            if (!container.isRunning()) {
                container.start();
                log.info("[Init] Kafka consumer {} manually started", kafkaGroupId);
            } else if (container.isPauseRequested()) {
                container.resume();
                log.info("[Init] Kafka consumer {} resumed", kafkaGroupId);
            }
        } else {
            log.info("[Init] Kafka consumer {} not started due to config disabled", kafkaGroupId);
        }

        lastState = enabled;
    }


    @Scheduled(fixedDelay = 5000)  // 每5秒检查一次配置
    public void checkAndControlConsumer() {
        boolean enabled = CommonConfigExpander.getFtVehicleDataEnable();

        if (lastState != null && lastState == enabled) {
            // 状态未变，跳过
            return;
        }

        MessageListenerContainer container = kafkaListenerEndpointRegistry.getListenerContainer(kafkaGroupId);
        if (container == null) {
            log.info("[Scheduler] no consumer {}", kafkaGroupId);
            return;
        }

        if (enabled) {
            if (!container.isRunning()) {
                container.start();
                log.info("[Scheduler] Kafka consumer {} started", kafkaGroupId);
            } else if (container.isPauseRequested()) {
                container.resume();
                log.info("[Scheduler] Kafka consumer {} resumed", kafkaGroupId);
            }
        } else {
            if (container.isRunning() && !container.isPauseRequested()) {
                container.pause();
                log.info("[Scheduler] Kafka consumer {} paused", kafkaGroupId);
            }
        }

        lastState = enabled;
    }
}
```

> kafkaListenerEndpointRegistry.getListenerContainer(kafkaGroupId) 指的是 KafkaListener 中的 id

