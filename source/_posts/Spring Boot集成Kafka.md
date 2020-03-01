---
title: Spring Boot 集成 Kafka
date: 2020-01-16 17:22:00
categories: Kafka
tags:
    - Spring Boot
    - Kafka
---
## Maven 依赖
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
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
    public static final String TOPIC = "topic";

    /**
     * 当主题不存在时才会创建新的主题
     */
    @Bean
    public NewTopic topic() {
        return TopicBuilder.name(TOPIC).partitions(1).replicas(1).build();
    }
}

@Slf4j
@Component
public class MessageSender {
    private final KafkaTemplate<Object, Object> kafkaTemplate;

    public MessageSender(KafkaTemplate<Object, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    /**
     * 简单发送消息
     */
    public void send(Message message) {
        kafkaTemplate.send(TOPIC, message);
    }
}
```

## 消费方
```java
@Configuration
public class KafkaConfiguration {
    public static final String TOPIC = "topic";

    /**
     * 当主题不存在时才会创建新的主题
     */
    @Bean
    public NewTopic topic() {
        return TopicBuilder.name(TOPIC).partitions(1).replicas(1).build();
    }
}

@Slf4j
@Component
public class MessageReceiver {

    /**
     * 接收消息，参数类型为 ConsumerRecord
     */
    @KafkaListener(id = "group-2-1", topics = {TOPIC})
    public void receive(ConsumerRecord<Object, Object> record, Acknowledgment acknowledgment) {
        Optional<Object> messageOptional = Optional.ofNullable(record.value());
        if (messageOptional.isPresent()) {
            Object message = messageOptional.get();
            log.info("Receive message: {}", message);
        }

        // 设置了手动 ack 模式时，需要在消息处理完毕之后，手动调用 acknowledge
        acknowledgment.acknowledge();
    }

    /**
     * 接收消息，参数类型为 String
     */
    @KafkaListener(id = "group-2-2", topics = {TOPIC})
    public void receive(String message, Acknowledgment acknowledgment) {
        log.info("Receive message: {}", message);
        acknowledgment.acknowledge();
    }
}
```

## 高级配置
### 序列化/反序列化
当我们发送对象类型消息数据时，在消费者方默认接收到的数据类型是字符串，如果我们希望接收到的是对象类型，可以在生产者/消费者两方都添加一点配置。
* 生产方
```yaml
spring:
  kafka:
    # 生产者
    producer:
      properties:
        # 建立映射：message 与 samples.dto.Message 对应
        spring.json.type.mapping: message:samples.dto.Message
```

* 消费方
```java
@Configuration
public class KafkaConfiguration {

    /**
     * 消息类型转换
     */
    @Bean
    public RecordMessageConverter converter() {
        DefaultJackson2JavaTypeMapper typeMapper = new DefaultJackson2JavaTypeMapper();
        typeMapper.setTypePrecedence(Jackson2JavaTypeMapper.TypePrecedence.TYPE_ID);
        typeMapper.addTrustedPackages("samples.dto");
        Map<String, Class<?>> mappings = new HashMap<>(10);
        mappings.put("message", Message.class);
        typeMapper.setIdClassMapping(mappings);

        StringJsonMessageConverter converter = new StringJsonMessageConverter();
        converter.setTypeMapper(typeMapper);
        return converter;
    }
}
```

之后，我们就能够像下面这样接收 Message 参数的消息了：
```java
/**
 * 接收消息，参数类型为 Message
 */
@KafkaListener(id = "group-2-3", topics = {TOPIC})
public void receive(Message message, Acknowledgment acknowledgment) {
    log.info("Receive message: {}", message);

    // 设置了手动 ack 模式时，需要在消息处理完毕之后，手动调用 acknowledge
    acknowledgment.acknowledge();
}
```

在后续演示代码中，默认都会配置序列化/反序列化，就不再一一列举了。

### 获取消息发送结果
在发送消息之后，我们可以获取到消息发送结果，既可以以同步方式获取结果，也可以以异步方式获取结果。
* 同步方式
```java
/**
 * 发送消息，同时同步获取消息发送结果
 */
public void sendAndGetResultSync(Message message) {
    ListenableFuture<SendResult<Object, Object>> future = kafkaTemplate.send(TOPIC, message);
    try {
        SendResult<Object, Object> result = future.get();
        log.info("Sync get send result success, result: {}", result);
    } catch (Throwable e) {
        log.error("Sync get send result failure", e);
    }
}
```

* 异步方式
```java
/**
 * 发送消息，同时异步获取消息发送结果
 */
public void sendAndGetResultAsync(Message message) {
    kafkaTemplate.send(TOPIC, message).addCallback(new ListenableFutureCallback<SendResult<Object, Object>>() {

        @Override
        public void onSuccess(SendResult<Object, Object> result) {
            log.info("Async get send result success, result: {}", result);

        }

        @Override
        public void onFailure(Throwable e) {
            log.error("Async get send result failure", e);
        }
    });
}
```

### 使用事务
#### 开启事务
当我们在生产者方配置了属性 transaction-id-prefix 后，Spring 会自动帮我们开启事务。
不过开启事务之后，retries 属性需要设置为大于 0，acks 属性需要设置为 all 或 -1。
另外，我们还需要将消费者方的 isolation.level 设置为 read_committed，这样对于未提交事务的消息，消费者就不会读取到。
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

#### 使用事务
有两种方式
Spring 提供了两种方式使用事务：调用 KafkaTemplate 的 executeInTransaction 方法，或使用 @Transactional。

* 调用 KafkaTemplate 的 executeInTransaction 方法
```java
/**
 * 发送消息，并开启事务，使用 kafkaTemplate.executeInTransaction
 */
public void sendInTransactionByMethod(Message message) {
    kafkaTemplate.executeInTransaction(kafkaTemplate -> {
        kafkaTemplate.send(TOPIC, message);
        log.info("Send in transaction by method success");
        //noinspection ConstantConditions
        return null;
        // throw new RuntimeException("fail");
    });
}
```

* 使用 @Transactional
```java
/**
 * 发送消息，并开启事务，使用 @Transactional
 */
@Transactional(rollbackFor = Exception.class)
public void sendInTransactionByAnnotation(Message message) {
    kafkaTemplate.send(TOPIC, message);
    log.info("Send in transaction by annotation success");
    // throw new RuntimeException("fail");
}
```

需要注意的是，在生产者开启事务之后，所有发送消息的地方都必须放在事务中执行。

### 转发消息
消费者在收到消息并对消息进行处理之后，可以再将新的消息发送出去。
在消费方，我们既可以使用 kafkaTemplate.send 实现手动发送消息，也可以使用 @Send 实现自动发送消息。
* 生产方
```java
@Configuration
public class KafkaConfiguration {
    public static final String TOPIC_RESEND = "topic-resend";
    public static final String TOPIC_RESEND_NEXT = "topic-resend-next";

    @Bean
    public NewTopic topicResend() {
        return TopicBuilder.name(TOPIC_RESEND).partitions(1).replicas(1).build();
    }

    @Bean
    public NewTopic topicResendNext() {
        return TopicBuilder.name(TOPIC_RESEND_NEXT).partitions(1).replicas(1).build();
    }
}

/**
 * 消息生产方同时作为消费者，接收其对应的消费者返回的消息
 */
@Slf4j
@Component
public class MessageReceiver {

    /**
     * 接收消息，参数类型为 ConsumerRecord
     */
    @KafkaListener(id = "group-1-1", topics = {TOPIC_RESEND_NEXT})
    public void receive(ConsumerRecord<Object, Object> record, Acknowledgment acknowledgment) {
        Optional<Object> messageOptional = Optional.ofNullable(record.value());
        if (messageOptional.isPresent()) {
            Object message = messageOptional.get();
            log.info("Receive message: {}", message);
        }
        acknowledgment.acknowledge();
    }

    /**
     * 接收消息，参数类型为 String
     */
    @KafkaListener(id = "group-1-2", topics = {TOPIC_RESEND_NEXT})
    public void receive(String msg, Acknowledgment acknowledgment) {
        log.info("Receive msg: {}", msg);
        acknowledgment.acknowledge();
    }
}

@Slf4j
@Component
public class MessageReceiver {

    /**
     * 接收消息，参数类型为 ConsumerRecord
     */
    @KafkaListener(id = "group-1-1", topics = {TOPIC_RESEND_NEXT})
    public void receive(ConsumerRecord<Object, Object> record, Acknowledgment acknowledgment) {
        Optional<Object> messageOptional = Optional.ofNullable(record.value());
        if (messageOptional.isPresent()) {
            Object message = messageOptional.get();
            log.info("Receive message: {}", message);
        }
        acknowledgment.acknowledge();
    }

    /**
     * 接收消息，参数类型为 String
     */
    @KafkaListener(id = "group-1-2", topics = {TOPIC_RESEND_NEXT})
    public void receive(String msg, Acknowledgment acknowledgment) {
        log.info("Receive msg: {}", msg);
        acknowledgment.acknowledge();
    }
}
```

* 消费方
```java
@Configuration
public class KafkaConfiguration {
    public static final String TOPIC_RESEND = "topic-resend";
    public static final String TOPIC_RESEND_NEXT = "topic-resend-next";

    @Bean
    public NewTopic topicResend() {
        return TopicBuilder.name(TOPIC_RESEND).partitions(1).replicas(1).build();
    }

    @Bean
    public NewTopic topicResendNext() {
        return TopicBuilder.name(TOPIC_RESEND_NEXT).partitions(1).replicas(1).build();
    }
}

@Slf4j
@Component
public class MessageReceiver {
    private final KafkaTemplate<Object, Object> kafkaTemplate;

    @SuppressWarnings("SpringJavaInjectionPointsAutowiringInspection")
    public MessageReceiver(KafkaTemplate<Object, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    /**
     * 接收消息，在处理完消息之后，再将新的消息 手动 重新发送出去
     */
    @SendTo
    @KafkaListener(id = "group-2-4", topics = {TOPIC_RESEND})
    public void receiveAndResendManual(Message message, Acknowledgment acknowledgment) {
        log.info("Receive message: {}", message);
        acknowledgment.acknowledge();
        kafkaTemplate.send(TOPIC_RESEND_NEXT, message.getMsg());
    }

    /**
     * 接收消息，在处理完消息之后，再将新的消息 自动 重新发送出去
     */
    @SendTo(TOPIC_RESEND_NEXT)
    @KafkaListener(id = "group-2-5", topics = {TOPIC_RESEND})
    public String receiveAndResendAuto(Message message, Acknowledgment acknowledgment) {
        log.info("Receive message: {}", message);
        acknowledgment.acknowledge();
        return message.getMsg();
    }
}
```

### 获取消息回复
生产者在发送消息之后，可以同时等待获取消费者接收并处理消息之后的回复，就像传统的 RPC 交互那样，要实现这个功能，我们需要使用 ReplyingKafkaTemplate。
ReplyingKafkaTemplate 是 KafkaTemplate 的一个子类，它除了继承父类的方法，还新增了方法 sendAndReceive ，该方法实现了消息发送/回复的语义。
Spring Boot 没有提供开箱即用的 ReplyingKafkaTemplate，我们需要做些额外的配置。

* 生产方
```java
@Configuration
public class KafkaConfiguration {
    public static final String TOPIC_RECEIVE = "topic-receive";

    public static final String TOPIC_REPLIES = "replies";
    public static final String GROUP_REPLIES = "repliesGroup";

    @Bean
    public NewTopic topicReceive() {
        return TopicBuilder.name(TOPIC_RECEIVE).partitions(1).replicas(1).build();
    }

    @Bean
    public NewTopic topicReplies() {
        return TopicBuilder.name(TOPIC_REPLIES).partitions(1).replicas(1).build();
    }

    @Bean
    public ConcurrentMessageListenerContainer<Object, Object> concurrentMessageListenerContainer(
            ConcurrentKafkaListenerContainerFactory<Object, Object> containerFactory) {
        ConcurrentMessageListenerContainer<Object, Object> container =
                containerFactory.createContainer(TOPIC_REPLIES);
        container.getContainerProperties().setGroupId(GROUP_REPLIES);
        container.setAutoStartup(false);
        return container;
    }

    @Bean
    public ReplyingKafkaTemplate<Object, Object, Object> replyingKafkaTemplate(
            ProducerFactory<Object, Object> producerFactory,
            ConcurrentMessageListenerContainer<Object, Object> concurrentMessageListenerContainer) {
        ReplyingKafkaTemplate<Object, Object, Object> kafkaTemplate =
                new ReplyingKafkaTemplate<>(producerFactory, concurrentMessageListenerContainer);
        // kafkaTemplate.setSharedReplyTopic(true);
        return kafkaTemplate;
    }

    /**
     * 如果不配置 kafkaTemplate，启动时会抛出循环依赖异常
     */
    @Bean
    public KafkaTemplate<Object, Object> kafkaTemplate(ProducerFactory<Object, Object> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }
}

@Slf4j
@Component
public class MessageSender {
    /**
     * KafkaTemplate 的子类，除了继承父类的方法之外，还提供了 sendAndReceive 方法，可以用来实现类似传统 RPC 那样的交互：即发送消息，并等待消息处理结果。
     */
    private final ReplyingKafkaTemplate<Object, Object, Object> replyingKafkaTemplate;

    public MessageSender(ReplyingKafkaTemplate<Object, Object, Object> replyingKafkaTemplate) {
        this.replyingKafkaTemplate = replyingKafkaTemplate;
    }

    /**
     * 发送消息，同时等待消息结果
     */
    public void sendAndReceive(Message message) {
        ProducerRecord<Object, Object> producerRecord = new ProducerRecord<>(TOPIC_RECEIVE, message);
        // producerRecord.headers().add(new RecordHeader(KafkaHeaders.REPLY_TOPIC, TOPIC_REPLIES.getBytes()));
        RequestReplyFuture<Object, Object, Object> replyFuture = replyingKafkaTemplate.sendAndReceive(producerRecord);
        ConsumerRecord<Object, Object> consumerRecord;
        try {
            consumerRecord = replyFuture.get();
            log.info("Receive reply success, result: {}", consumerRecord.value());
        } catch (InterruptedException | ExecutionException e) {
            log.error("Receive reply failure", e);
        }
    }
}
```

* 消费方
```java
@Configuration
public class KafkaConfiguration {
    public static final String TOPIC_RECEIVE = "topic-receive";

    @Bean
    public NewTopic topicReceive() {
        return TopicBuilder.name(TOPIC_RECEIVE).partitions(1).replicas(1).build();
    }
}

@Slf4j
@Component
public class MessageReceiver {

    /**
     * 接收消息，在处理完消息之后，将处理结果返回给生产方
     */
    @SendTo
    @KafkaListener(id = "group-2-6", topics = {TOPIC_RECEIVE})
    public String receiveAndReply(Message message, Acknowledgment acknowledgment) {
        log.info("Receive message: {}", message);
        acknowledgment.acknowledge();
        return "successful";
    }
}
```

### 多方法处理消息
组合使用 @KafkaListener 和 @KafkaHandler，能够让我们在传递消息时，根据转换后的消息有效负载类型来确定调用哪个方法。

* 生产方
```java
@Configuration
public class KafkaConfiguration {
    public static final String TOPIC_MULTIPLE = "topic-multiple";

    @Bean
    public NewTopic topicMultiple() {
        return TopicBuilder.name(TOPIC_MULTIPLE).partitions(1).replicas(1).build();
    }
}

@Slf4j
@Component
public class MessageSender {
    private final KafkaTemplate<Object, Object> kafkaTemplate;

    public MessageSender(KafkaTemplate<Object, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    /**
     * 发送消息，消息类型为字符串，消费者根据消息类型决定执行哪个方法
     */
    public void sendToMultipleHandlers(String msg) {
        kafkaTemplate.send(TOPIC_MULTIPLE, msg);
    }

    /**
     * 发送消息，消息类型为 Message，消费者根据消息类型决定执行哪个方法
     */
    public void sendToMultipleHandlers(Message message) {
        kafkaTemplate.send(TOPIC_MULTIPLE, message);
    }
}
```

* 消费方
```java
@Configuration
public class KafkaConfiguration {
    public static final String TOPIC_MULTIPLE = "topic-multiple";

    @Bean
    public NewTopic topicMultiple() {
        return TopicBuilder.name(TOPIC_MULTIPLE).partitions(1).replicas(1).build();
    }
}

@Slf4j
@KafkaListener(id = "group-2-7", topics = {TOPIC_MULTIPLE})
@Component
public class MessageReceiverMultipleMethods {

    /**
     * 处理字符串类型的消息
     */
    @KafkaHandler
    public void handlerStr(String message, Acknowledgment acknowledgment) {
        log.info("Receive message: {}, type of String", message);
        acknowledgment.acknowledge();
    }

    /**
     * 处理 Message 类型的消息
     */
    @KafkaHandler
    public void handlerMessage(Message message, Acknowledgment acknowledgment) {
        log.info("Receive message: {}, type of Message", message);
        acknowledgment.acknowledge();
    }

    /**
     * 处理其它类型的消息
     */
    @KafkaHandler(isDefault = true)
    public void handlerUnknown(Object message, Acknowledgment acknowledgment) {
        log.info("Receive message: {}, type of Unknown", message);
        acknowledgment.acknowledge();
    }
}
```

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

    @KafkaListener(id = "group-2-1", topics = {"topic-1"})
    public void receive(ConsumerRecord<Object, Object> record, Acknowledgment acknowledgment) {
        log.info("receive record: {}", record);
        Optional<Object> messageOptional = Optional.ofNullable(record.value());
        if (messageOptional.isPresent()) {
            Object message = messageOptional.get();
            log.info("receive message: {}", message);
        }
        acknowledgment.acknowledge();
    }
}
```

### 异常处理
对于消费者在处理消息过程中抛出的异常，我们可以设置 errorHandler，然后在 errorHandler 中统一处理。

```java
/**
 * 接收消息，处理消息时抛出异常，异常由 errorHandler 进行处理
 */
@KafkaListener(id = "group-2-8", topics = {TOPIC_EXCEPTION}, errorHandler = "customErrorHandler")
public void receiveWithException(Message message, Acknowledgment acknowledgment) {
    log.info("Receive message: {}", message);
    throw new RuntimeException("error");
}

@Slf4j
@Service("customErrorHandler")
public class CustomKafkaListenerErrorHandler implements KafkaListenerErrorHandler {
    @Override
    public Object handleError(Message<?> message, ListenerExecutionFailedException exception) {
        log.error("Handle message with exception, message: {}", message.getPayload().toString());
        return null;
    }

    @Override
    public Object handleError(Message<?> message, ListenerExecutionFailedException exception, Consumer<?, ?> consumer) {
        log.error("Handle message with exception, message: {}", message.getPayload().toString());
        return null;
    }
}
```

### 并发接收消息
```java
 /**
 * 并发接收消息
 */
@KafkaListener(id = "group-2-9", topics = {TOPIC_CONCURRENT}, concurrency = "3")
public void receiveConcurrent(Message message, Acknowledgment acknowledgment) {
    log.info("Receive message: {}", message);
    acknowledgment.acknowledge();
}
```

### 暂停与恢复消费
通过使用 KafkaListenerEndpointRegistry，我们可以动态的暂停与恢复消费者消费消息。

```java
/**
 * bean 的名称需是类名，不然会找不到 bean
 */
private final KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

public MessageSender(KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry) {
    this.kafkaListenerEndpointRegistry = kafkaListenerEndpointRegistry;
}

/**
 * 指定消费者开始消费
 */
public void startListener(String listenerId) {
    kafkaListenerEndpointRegistry.getListenerContainer(listenerId).start();
}

/**
 * 指定消费者暂停消费
 */
public void stopListener(String listenerId) {
    kafkaListenerEndpointRegistry.getListenerContainer(listenerId).pause();
}

/**
 * 指定消费者恢复消费
 */
public void resumeListener(String listenerId) {
    kafkaListenerEndpointRegistry.getListenerContainer(listenerId).resume();
}
```

### 消息重试与死信队列
当消费者在处理消息过程中发生异常时，我们可以进行多次重试，如果最终还是存在异常，我们可以将消息发送到预定的 Topic，即死信队列中。

```java
/**
 * 在消费方配置 errorHandler
 */
@Bean
public ConcurrentKafkaListenerContainerFactory<Object, Object> kafkaListenerContainerFactory(
        ConcurrentKafkaListenerContainerFactoryConfigurer configurer,
        ConsumerFactory<Object, Object> consumerFactory,
        KafkaTemplate<Object, Object> kafkaTemplate) {
    ConcurrentKafkaListenerContainerFactory<Object, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();
    configurer.configure(factory, consumerFactory);
    factory.setErrorHandler(new SeekToCurrentErrorHandler(new DeadLetterPublishingRecoverer(kafkaTemplate), new FixedBackOff(0L, 3L)));
    return factory;
}

/**
 * 当消费者处理消息时发生异常，且多次重试后依然异常，那么消息会被发送到死信队列
 * 这里的消息类型参数不能是 Message 类型，不然不会进入到该方法中
 */
@KafkaListener(id = "group-2-10-2", topics = {TOPIC_DEAD_LETTER + ".DLT"})
public void receiveByDeadLetter(ConsumerRecord<Object, Object> record, Acknowledgment acknowledgment) {
    log.info("Receive message from DLT, message: {}", record.value());
    acknowledgment.acknowledge();
}
```

### 拦截器
Apache Kafka 提供了一种向生产者和消费者添加拦截器的机制。下面我们将演示如何在 Spring Boot 中配置拦截器。

* 生产方
```java
@Configuration
public class KafkaConfiguration {
    @Bean
    public ProducerFactory<Object, Object> kafkaProducerFactory(KafkaProperties properties, CustomComponent customComponent) {
        Map<String, Object> producerProperties = properties.buildProducerProperties();
        producerProperties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, CustomProducerInterceptor.class.getName());
        producerProperties.put("custom.component", customComponent);
        DefaultKafkaProducerFactory<Object, Object> factory = new DefaultKafkaProducerFactory<>(producerProperties);
        String transactionIdPrefix = properties.getProducer().getTransactionIdPrefix();
        if (transactionIdPrefix != null) {
            factory.setTransactionIdPrefix(transactionIdPrefix);
        }
        return factory;
    }
}

@Slf4j
@Component
public class CustomComponent {

    public void doSomething() {
        log.info("Do something");
    }
}

@Slf4j
public class CustomProducerInterceptor implements ProducerInterceptor<Object, Object> {
    private CustomComponent customComponent;

    @Override
    public ProducerRecord<Object, Object> onSend(ProducerRecord<Object, Object> record) {
        log.info("Before send message, do your own business");
        customComponent.doSomething();
        return record;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {

    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {
        this.customComponent = (CustomComponent) configs.get("custom.component");
    }
}
```

* 消费方
```java
@Configuration
public class KafkaConfiguration {
    @Bean
    public ProducerFactory<Object, Object> kafkaProducerFactory(KafkaProperties properties, CustomComponent customComponent) {
        Map<String, Object> producerProperties = properties.buildProducerProperties();
        producerProperties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, CustomProducerInterceptor.class.getName());
        producerProperties.put("custom.component", customComponent);
        DefaultKafkaProducerFactory<Object, Object> factory = new DefaultKafkaProducerFactory<>(producerProperties);
        String transactionIdPrefix = properties.getProducer().getTransactionIdPrefix();
        if (transactionIdPrefix != null) {
            factory.setTransactionIdPrefix(transactionIdPrefix);
        }
        return factory;
    }
}

@Slf4j
@Component
public class CustomComponent {

    public void doSomething() {
        log.info("Do something");
    }
}

@Slf4j
public class CustomProducerInterceptor implements ProducerInterceptor<Object, Object> {
    private CustomComponent customComponent;

    @Override
    public ProducerRecord<Object, Object> onSend(ProducerRecord<Object, Object> record) {
        log.info("Before send message, do your own business");
        customComponent.doSomething();
        return record;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {

    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {
        this.customComponent = (CustomComponent) configs.get("custom.component");
    }
}
```


