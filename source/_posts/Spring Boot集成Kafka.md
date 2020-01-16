---
title: Spring Boot集成Kafka
date: 2020-01-16 17:22:00
categories: Kafka
tags:
    - Spring Boot
---
## Maven 依赖
```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
    </dependencies>
```

## 配置属性
```yaml
spring:
  kafka:
    # kafka 集群地址
    bootstrap-servers: localhost:9092,localhost:9093,localhost:9094
    # 生产者
    producer:
      # 生产者每次批量发送数据的大小，默认值 16K
      batch-size: 16384
      # 生产者能够使用的缓存的大小，默认值 32M
      buffer-memory: 33554432
      # 生产者发送数据失败时重试的次数，默认值 0
      retries: 0
      # 指定了必须要有多少个分区副本收到消息，生产者才会认为写入消息是成功的，这个参数对消息丢失的可能性有重大影响。默认值 1
      # 0：生产者往集群发送数据不需要等到集群的返回，不确保消息发送成功。安全性最低但是效率最高。
      # 1：生产者往集群发送数据只要 Leader 应答就可以发送下一条，只确保 Leader 接收成功。
      # all或-1：往集群发送数据需要所有的 ISR Follower 都完成从 Leader 的同步才会发送下一条，确保 Leader 发送成功和所有的副本都成功接收。安全性最高，但是效率最低。
      acks: 1
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      properties:
        # 一个 batch 被创建之后，最多等待多久被发送出去，默认值 0
        # 配合 batch-size 一起使用：满足 batch.size 和 ling.ms 之一，生产者便开始发送 batch
        linger.ms: 0
    # 消费者
    consumer:
      # 用来唯一标识 consumer 进程所在组的字符串，如果设置同样的 group id，表示这些 processes 都是属于同一个 consumer group
      group-id: group-1
      # 如果为真，消费者所 fetch 的消息的 offset 将会自动的同步到 zookeeper
      enable-auto-commit: true
      # 消费者向 zookeeper 提交 offset 的频率，当 enable-auto-commit 为 true 时生效，默认值 5s
      auto-commit-interval: 5000
      # earliest：当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
      # latest：当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据
      # none: 当各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

## 生产方
```java
@Data
public class Message {
    private Long id;
    private String msg;
    private Date sendTime;
}

@Configuration
public class KafkaConfiguration {

    @Bean
    public NewTopic topic1() {
        return TopicBuilder.name("topic-1").partitions(1).replicas(1).build();
    }
}

@Slf4j
@Component
public class MessageSender {
    private final KafkaTemplate<String, String> kafkaTemplate;

    @SuppressWarnings("SpringJavaInjectionPointsAutowiringInspection")
    public MessageSender(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void send() throws JsonProcessingException {
        Message message = new Message();
        message.setId(System.currentTimeMillis());
        message.setMsg(UUID.randomUUID().toString());
        message.setSendTime(new Date());
        String messageStr = new ObjectMapper().writeValueAsString(message);
        log.info("send message: {}", messageStr);
        kafkaTemplate.send("topic-1", messageStr);
    }
}

@SpringBootApplication
public class ProducerApplication implements CommandLineRunner {
    private final MessageSender messageSender;

    public ProducerApplication(MessageSender messageSender) {
        this.messageSender = messageSender;
    }

    public static void main(String[] args) {
        SpringApplication.run(ProducerApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        messageSender.send();
    }
}
```

## 消费方
```java
@Configuration
public class KafkaConfiguration {

    @Bean
    public NewTopic topic1() {
        return TopicBuilder.name("topic-1").partitions(1).replicas(1).build();
    }
}

@Slf4j
@Component
public class MessageReceiver {

    @KafkaListener(topics = {"topic-1"})
    public void receive(ConsumerRecord<?, ?> record) {
        log.info("receive record: {}", record);
        Optional<?> messageOptional = Optional.ofNullable(record.value());
        if (messageOptional.isPresent()) {
            Object message = messageOptional.get();
            log.info("receive message: {}", message);
        }
    }
}

@SpringBootApplication
public class ConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```