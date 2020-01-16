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
      group-id: group-x
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
    private final KafkaTemplate<Object, Object> kafkaTemplate;

    @SuppressWarnings("SpringJavaInjectionPointsAutowiringInspection")
    public MessageSender(KafkaTemplate<Object, Object> kafkaTemplate) {
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

    @KafkaListener(id = "group-2-1", topics = {"topic-1"})
    public void receive(ConsumerRecord<Object, Object> record) {
        log.info("receive record: {}", record);
        Optional<Object> messageOptional = Optional.ofNullable(record.value());
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

## 高级配置
### 多方法处理消息
组合使用 @KafkaListener 和 @KafkaHandler，能够让我们在传递消息时，根据转换后的消息有效负载类型来确定调用哪个方法。如下：
```java
@Slf4j
@KafkaListener(id = "group-2-2", topics = {"topic-1"})
@Component
public class MessageReceiverMultipleMethods {

    @KafkaHandler
    public void handlerStr(String message, Acknowledgment acknowledgment) {
        log.info("receive message: {}, type of String", message);
        acknowledgment.acknowledge();
    }

    @KafkaHandler
    public void handlerMessage(Message message, Acknowledgment acknowledgment) {
        log.info("receive message: {}, type of Message", message);
        acknowledgment.acknowledge();
    }

    @KafkaHandler(isDefault = true)
    public void handlerUnknown(Object message, Acknowledgment acknowledgment) {
        log.info("receive message: {}, type of Unknown", message);
        acknowledgment.acknowledge();
    }
}
```
当生产者发送字符串类型消息时，会执行方法 handlerStr；当生产者发送 Message 类型消息时，会执行方法 handlerMessage；当生产者发送其它类型（如整形）消息时，则会执行方法 handlerUnknown。

当然，要将消息映射到不同的类型上，我们需要做一点额外的配置：
* 修改 application.yml
```yaml
spring:
  kafka:
    # 生产者
    producer:
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      spring.json.type.mapping: message:samples.Message
```
* 配置 RecordMessageConverter Bean
```java
@Bean
public RecordMessageConverter converter() {
    DefaultJackson2JavaTypeMapper typeMapper = new DefaultJackson2JavaTypeMapper();
    typeMapper.setTypePrecedence(Jackson2JavaTypeMapper.TypePrecedence.TYPE_ID);
    typeMapper.addTrustedPackages("samples");
    Map<String, Class<?>> mappings = new HashMap<>(1);
    mappings.put("message", Message.class);
    typeMapper.setIdClassMapping(mappings);

    StringJsonMessageConverter converter = new StringJsonMessageConverter();
    converter.setTypeMapper(typeMapper);
    return converter;
}
```

### 开启事务
使用 kafka 事务，我们能够保证生产者发送到多个分区的消息要么都成功要么都失败。Spring 提供了两种方式使用事务：
* 使用 @Transactional
* 调用 KafkaTemplate 的 executeInTransaction 方法

以下代码演示了如何使用事务：
```java
@Slf4j
@Component
public class MessageSenderTransaction {
    private final KafkaTemplate<Object, Object> kafkaTemplate;

    @SuppressWarnings("SpringJavaInjectionPointsAutowiringInspection")
    public MessageSenderTransaction(KafkaTemplate<Object, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendByInvokeMethod() {
        kafkaTemplate.executeInTransaction(kafkaTemplate -> {
            send();

            //noinspection ConstantConditions
            return null;
//            throw new RuntimeException("fail");
        });
    }

    @Transactional(rollbackFor = Exception.class)
    public void sendWithAnnotation() {
        send();
//        throw new RuntimeException("fail");
    }

    private void send() {
        Message message = new Message();
        message.setId(System.currentTimeMillis());
        message.setMsg(UUID.randomUUID().toString());
        message.setSendTime(new Date());
        log.info("send message: {}", message);
        kafkaTemplate.send("topic-1", message);
    }
}
```

当然，要想使用事务，我们需要做一点额外的配置：
* 修改 application.yml
```yaml
spring:
  kafka:
    # 生产者
    producer:
      # 生产者发送数据失败时重试的次数，默认值 0
      retries: 3
      # 指定了必须要有多少个分区副本收到消息，生产者才会认为写入消息是成功的，这个参数对消息丢失的可能性有重大影响。默认值 1
      # 0：生产者往集群发送数据不需要等到集群的返回，不确保消息发送成功。安全性最低但是效率最高。
      # 1：生产者往集群发送数据只要 Leader 应答就可以发送下一条，只确保 Leader 接收成功。
      # all或-1：往集群发送数据需要所有的 ISR Follower 都完成从 Leader 的同步才会发送下一条，确保 Leader 发送成功和所有的副本都成功接收。安全性最高，但是效率最低。
      acks: -1
      # 事务 id 前缀，设置该属性后会开启事务
      transaction-id-prefix: tx.
    # 消费者
    consumer:
      properties:
        # 事务隔离级别（read_committed 和 read_uncommitted）
        isolation.level: read_committed
```
如上所示，首先需要添加配置`transaction-id-prefix: tx.`，然后需要将`retries`的值设置为大于 0，并将`acks`的值设置为 all 或 -1。
另外需要注意的是，生产者开启事务之后，所有发送消息的地方都必须放在事务中执行。

### 手动提交 offset （ack）
默认情况下，Kafka 会自动帮我们提交 offset，但是这样做容易导致消息重复消费或消失丢失：
* 在消费者收到消息之后，且 kafka 未自动提交 offset 之前，broker 宕机了，然后重启 broker，此时消费者会从原来的 offset 开始消费，于是出现了重复消费；
* 在消费者收到消息之后，且消费者还没有处理完消息时，由于自动提交的间隔时间到了，于是 kafka 自动提交了 offset，但是之后消费者又挂掉了，那么当消费者重启之后，会从下一个 offset 开始消费，这样前面的消息就丢失了。
我们可以改为使用手动提交 offset，只需要做两处调整：
1. 修改 application.yml
```yaml
spring:
  kafka:
    # 消费方
    consumer:
      enable-auto-commit: false
    listener:
      # 手动 ack，默认值 batch
      ack-mode: manual
```
2. 消息处理完成之后，调用 Acknowledgment 的 acknowledge 方法
```java
@Slf4j
@Component
public class MessageReceiver {

    @KafkaListener(topics = {"topic-1"})
    public void receive(ConsumerRecord<String, String> record, Acknowledgment acknowledgment) {
        log.info("receive record: {}", record);
        Optional<String> messageOptional = Optional.ofNullable(record.value());
        if (messageOptional.isPresent()) {
            String message = messageOptional.get();
            log.info("receive message: {}", message);
        }
        acknowledgment.acknowledge();
    }
}
```

