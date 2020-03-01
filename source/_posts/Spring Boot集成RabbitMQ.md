---
title: Spring Boot 集成 RabbitMQ
date: 2020-01-27 20:01:00
categories: RabbitMQ
tags:
    - Spring Boot
    - RabbitMQ
---
## Maven 依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

## 配置属性
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
```

## 配置类
```java
@Configuration
public class RabbitConfiguration {
    public static final String QUEUE = "queue";

    public static final String DIRECT_EXCHANGE = "direct-exchange";
    public static final String DIRECT_QUEUE = "direct-queue";
    public static final String DIRECT_ROUTING_KEY = "direct-routing-key";

    public static final String TOPIC_EXCHANGE = "topic-exchange";
    public static final String TOPIC_QUEUE = "topic-queue";
    public static final String TOPIC_ROUTING_KEY = "topic-routing-key.*";

    public static final String FANOUT_EXCHANGE = "fanout-exchange";
    public static final String FANOUT_QUEUE = "fanout-queue";

    public static final String HEADERS_EXCHANGE = "headers-exchange";
    public static final String HEADERS_QUEUE = "headers-queue";
    public static final String HEADERS_HEADER_KEY = "headers-header";

    public static final String CUSTOM_EXCHANGE = "custom-exchange";
    public static final String CUSTOM_QUEUE = "custom-queue";
    public static final String CUSTOM_ROUTING_KEY = "custom-routing-key";

    @Bean
    public Queue queue() {
        return new Queue(QUEUE);
    }

    @Bean
    public DirectExchange directExchange() {
        return new DirectExchange(DIRECT_EXCHANGE);
    }

    @Bean
    public Queue directQueue() {
        return new Queue(DIRECT_QUEUE);
    }

    @Bean
    public Binding directQueueBinding() {
        return BindingBuilder.bind(directQueue()).to(directExchange()).with(DIRECT_ROUTING_KEY);
    }

    @Bean
    public TopicExchange topicExchange() {
        return new TopicExchange(TOPIC_EXCHANGE);
    }

    @Bean
    public Queue topicQueue() {
        return new Queue(TOPIC_QUEUE);
    }

    @Bean
    public Binding topicQueueBinding() {
        return BindingBuilder.bind(topicQueue()).to(topicExchange()).with(TOPIC_ROUTING_KEY);
    }

    @Bean
    public FanoutExchange fanoutExchange() {
        return new FanoutExchange(FANOUT_EXCHANGE);
    }

    @Bean
    public Queue fanoutQueue() {
        return new Queue(FANOUT_QUEUE);
    }

    @Bean
    public Binding fanoutQueueBinding() {
        return BindingBuilder.bind(fanoutQueue()).to(fanoutExchange());
    }

    @Bean
    public HeadersExchange headersExchange() {
        return new HeadersExchange(HEADERS_EXCHANGE);
    }

    @Bean
    public Queue headersQueue() {
        return new Queue(HEADERS_QUEUE);
    }

    @Bean
    public Binding headersQueueBinding() {
        return BindingBuilder.bind(headersQueue()).to(headersExchange()).where(HEADERS_HEADER_KEY).exists();
    }
}
```

## 生产方
```java
@Slf4j
@Component
public class MessageSender {
    private final RabbitTemplate rabbitTemplate;

    public MessageSender(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void send(String message) {
        rabbitTemplate.convertAndSend(QUEUE, message);
    }

    public void sendToDirectQueue(String message) {
        rabbitTemplate.convertAndSend(DIRECT_EXCHANGE, DIRECT_ROUTING_KEY, message);
    }

    public void sendToTopicQueue(String message) {
        String keyPrefix = TOPIC_ROUTING_KEY.substring(0, TOPIC_ROUTING_KEY.length() - 1);
        rabbitTemplate.convertAndSend(TOPIC_EXCHANGE, keyPrefix + "xx", message);
    }

    public void sendToFanoutQueue(String message) {
        rabbitTemplate.convertAndSend(FANOUT_EXCHANGE, null, message);
    }

    public void sendToHeadersQueue(String message) {
        MessageProperties properties = new MessageProperties();
        properties.setContentType("UTF-8");
        properties.getHeaders().put(HEADERS_HEADER_KEY, "header-xx");

        Message messageObj = new Message(message.getBytes(), properties);
        rabbitTemplate.send(HEADERS_EXCHANGE, null, messageObj);
    }
}
```

## 消费方
```java
@Slf4j
@Component
public class MessageReceiver {

    @RabbitListener(queues = QUEUE)
    public void receive(String message) {
        log.info("Receive message: {}", message);
    }

    @RabbitListener(queues = DIRECT_QUEUE)
    public void receiveByDirectQueue(String message) {
        log.info("Receive message by direct-queue: {}", message);
    }

    @RabbitListener(queues = TOPIC_QUEUE)
    public void receiveByTopicQueue(String message) {
        log.info("Receive message by topic-queue: {}", message);
    }

    @RabbitListener(queues = FANOUT_QUEUE)
    public void receiveByFanoutQueue(String message) {
        log.info("Receive message by fanout-queue: {}", message);
    }

    @RabbitListener(queues = HEADERS_QUEUE)
    public void receiveByHeadersQueue(String message) {
        log.info("Receive message by headers-queue: {}", message);
    }
}
```

## 高级配置
### 序列化/反序列化
当我们发送对象类型消息数据时，在消费者方默认接收到的数据类型是字符串，如果我们希望接收到的是对象类型，可以在生产者/消费者两方都添加以下配置：
```java
@Bean
public Jackson2JsonMessageConverter jsonMessageConverter() {
    Jackson2JsonMessageConverter jsonConverter = new Jackson2JsonMessageConverter();
    jsonConverter.setClassMapper(classMapper());
    return jsonConverter;
}

@Bean
public DefaultClassMapper classMapper() {
    DefaultClassMapper classMapper = new DefaultClassMapper();
    Map<String, Class<?>> idClassMapping = new HashMap<>(1);
    idClassMapping.put("message", Message.class);
    classMapper.setIdClassMapping(idClassMapping);
    return classMapper;
}
```

之后，我们就能够像下面这样发送/接收 Message 参数的消息了：
```java
public void sendObject(samples.dto.Message message) {
    rabbitTemplate.convertAndSend(OBJECT_QUEUE, message);
}

@RabbitListener(queues = QUEUE)
public void receive(Message message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
    log.info("Receive message: {}", message);
    try {
        channel.basicAck(tag, false);
    } catch (IOException e) {
        log.error("Confirm message error", e);
    }
}
```

### 使用事务
1. 设置 RabbitTemplate
```
rabbitTemplate.setChannelTransacted(true);
```

2. 注入 RabbitTransactionManager Bean
```java
@Bean("rabbitTransactionManager")
public RabbitTransactionManager rabbitTransactionManager(CachingConnectionFactory connectionFactory) {
    return new RabbitTransactionManager(connectionFactory);
}
```

3. 添加 @Transactional
```java
/**
 * 发送消息，并开启事务，使用 @Transactional
 */
@Transactional(rollbackFor = Exception.class)
public void sendInTransactionByAnnotation(String message) {
    rabbitTemplate.convertAndSend(QUEUE, message);
    // 与 Kafka 的是，即使后面抛出异常，下面这条日志还是会输出
    log.info("Send in transaction by annotation success");
    throw new RuntimeException("fail");
}
```

需要注意的是，在生产者开启事务之后，属性 publisher-confirm-type 要设置为 NONE （默认）。

### 获取消息回复
生产者在发送消息之后，可以同时等待获取消费者接收并处理消息之后的回复，就像传统的 RPC 交互那样。

* 生产方
```java
/**
 * 发送消息，同时等待消息结果
 */
public void sendAndReceive(String message) {
    Object result = rabbitTemplate.convertSendAndReceive(RECEIVE_QUEUE, message);
    log.info("Receive reply success, result: {}", result);
}
```

* 消费方
```java
/**
 * 接收消息，在处理完消息之后，将处理结果返回给生产方
 */
@RabbitListener(queues = RECEIVE_QUEUE)
public String receiveAndReply(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
    log.info("Receive message: {}", message);
    try {
        channel.basicAck(tag, false);
    } catch (IOException e) {
        log.error("Confirm message error", e);
    }
    return "ok";
}
```

### 多方法处理消息
组合使用 @RabbitListener 和 @RabbitHandler，能够让我们在传递消息时，根据转换后的消息有效负载类型来确定调用哪个方法。

* 生产方
```java
/**
 * 发送消息，消息类型为字符串，消费者根据消息类型决定执行哪个方法
 */
public void sendToMultipleHandlers(String message) {
    rabbitTemplate.convertSendAndReceive(MULTIPLE_QUEUE, message);
}

/**
 * 发送消息，消息类型为 Message，消费者根据消息类型决定执行哪个方法
 */
public void sendToMultipleHandlers(samples.dto.Message message) {
    rabbitTemplate.convertSendAndReceive(MULTIPLE_QUEUE, message);
}
```

* 消费方
```java
@Slf4j
@RabbitListener(queues = MULTIPLE_QUEUE)
@Component
public class MessageReceiverMultipleMethods {

    /**
     * 处理字符串类型的消息
     */
    @RabbitHandler
    public void handlerStr(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("Receive message: {}, type of String", message);
        try {
            channel.basicAck(tag, false);
        } catch (IOException e) {
            log.error("Confirm message error", e);
        }
    }

    /**
     * 处理 Message 类型的消息
     */
    @RabbitHandler
    public void handlerMessage(Message message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("Receive message: {}, type of Message", message);
        try {
            channel.basicAck(tag, false);
        } catch (IOException e) {
            log.error("Confirm message error", e);
        }
    }

    /**
     * 处理其它类型的消息
     */
    @RabbitHandler(isDefault = true)
    public void handlerUnknown(Object message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        log.info("Receive message: {}, type of Unknown", message);
        try {
            channel.basicAck(tag, false);
        } catch (IOException e) {
            log.error("Confirm message error", e);
        }
    }
}
```

### 异常处理
对于消费者在处理消息过程中抛出的异常，我们可以设置 errorHandler，然后在 errorHandler 中统一处理。

```java
/**
 * 接收消息，处理消息时抛出异常，异常由 errorHandler 进行处理
 */
@RabbitListener(queues = EXCEPTION_QUEUE, errorHandler = "customErrorHandler")
public void receiveWithException(Message message) {
    log.info("Receive message: {}", message);
    throw new RuntimeException("error");
}

@Slf4j
@Service("customErrorHandler")
public class CustomRabbitListenerErrorHandler implements RabbitListenerErrorHandler {

    @Override
    public Object handleError(Message amqpMessage, org.springframework.messaging.Message<?> message, ListenerExecutionFailedException exception) {
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
@RabbitListener(queues = CONCURRENT_QUEUE, concurrency = "3")
public void receiveConcurrent(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
    log.info("Receive message: {}", message);
    try {
        channel.basicAck(tag, false);
    } catch (IOException e) {
        log.error("Confirm message error", e);
    }
}
```