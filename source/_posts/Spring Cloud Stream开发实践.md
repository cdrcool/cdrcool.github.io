---
title: Spring Cloud Stream 开发实践
date: 2020-03-14 19:35:00
categories: Spring Cloud
tags:
    - Spring Cloud Stream
---
## Maven 依赖
```xml
<!-- 消息中间件使用 RabbitMQ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

## 基于 Spring Cloud Function 的支持
### 基本示例
StreamApplication.java
```java
@Slf4j
@SpringBootApplication
public class StreamApplication {

    public static void main(String[] args) {
        SpringApplication.run(StreamApplication.class, args);
    }

    @Bean
    public Consumer<Person> consume() {
        return person -> log.info("Data received: {}", person);
    }

    @Data
    public static class Person {
        private String name;
    }
}
```

应用程序启动之后，转到 RabbitMQ 管理控制台（localhost:15672/），并向 consume-in-0.anonymous.X4taA71HT0yGbcyc7E06_Q 发送一条消息：
```json
{"name":"Sam Spade"}
```

然后，在控制台中我们就会看到以下输出：

```text
Receive message: StreamApplication.Person(name=Sam Spade)
```

### 消息转发
StreamApplication.java
```java
@Slf4j
@SpringBootApplication
public class StreamApplication {

    public static void main(String[] args) {
        SpringApplication.run(StreamApplication.class, args);
    }

    @Bean
    public Function<String, String> transform() {
        return payload -> {
            log.info("Data received before transform: {}", payload);
            return payload.toUpperCase();
        };
    }

    static class TestSource {
        private AtomicBoolean semaphore = new AtomicBoolean(true);

        @Bean
        public Supplier<String> send() {
            return () -> semaphore.getAndSet(!semaphore.get()) ? "foo" : "bar";

        }
    }

    static class TestSink {

        @Bean
        public Consumer<String> receive() {
            return payload -> log.info("Data received after transform: {}", payload);
        }
    }
}
```

application.yml
```yaml
spring:
  cloud:
    stream:
      function:
        definition: transform;send;receive
      bindings:
        transform-in-0:
          destination: testtock
        transform-out-0:
          destination: xformed
        send-out-0:
          destination: testtock
        receive-in-0:
          destination: xformed
```

应用程序启动之后，默认会每秒发送一条消息到 testtock，消息经转换后再发送到 xformed。控制台输出如下所示：

```text
Origin data received: foo
Transformed data received: FOO
Origin data received: bar
Transformed data received: BAR
...
```

## 基于注解的支持（遗留）
### 基本示例
#### 消费者
ConsumerApplication.java
```java
@Slf4j
@SpringBootApplication
public class ConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(Consumer2Application.class, args);
    }
}
```

MessageReceiver.java
```java
@Slf4j
@EnableBinding({Sink.class})
@Component
public class MessageReceiver {

    /**
     * 接收消息
     *
     * @param message 消息
     */
    @StreamListener(Processor.INPUT)
    public void handleMessage(String message) {
        log.info("Message received: " + message);
    }
}
```

application.yml
```yaml
server:
  port: 8081

spring:
  cloud:
    stream:
      bindings:
        input:
          # 对应 RabbitMQ 中的 Exchange（要与 output.destination 保持一致）
          destination: default-topic
          # 对应 RabbitMQ 中的 Queue
          group: default-topic-group
```

#### 生产者
ProducerApplication.java
```java
@Slf4j
@SpringBootApplication
public class ProducerApplication implements CommandLineRunner {
    @Autowired
    private MessageSender messageSender;

    public static void main(String[] args) {
        SpringApplication.run(ProducerApplication.class, args);
    }

    @Override
    public void run(String... args) {
        for (int i = 0; i < 10; i++) {
            messageSender.sendMessage(MessageBuilder.withPayload(String.format("Hello World, {201%s}!", i)).build());
        }
    }
}
```

MessageSender.java
```java
@EnableBinding(Source.class)
@Component
public class MessageSender {
    @Autowired
    private Source source;

    public void sendMessage(Message<String> message) {
        source.output().send(message);
    }
}
```

application.yml
```yaml
server:
  port: 8080

spring:
  cloud:
    stream:
      bindings:
        output:
          # 对应 RabbitMQ 中的 Exchange（要与 input.destination 保持一致）
          destination: default-topic
```

### 消息转发
消费者接收到消息之后，可以将处理后的消息转发给其他消费者。

#### 消费者
TransformSource.java
```java
public interface TransformSource {

    /**
     * Name of the output channel.
     */
    String OUTPUT = "transform-output";

    /**
     * @return output channel
     */
    @Output(TransformSource.OUTPUT)
    MessageChannel output();
}
```

TransformSink.java
```java
public interface TransformSink {

    /**
     * Input channel name.
     */
    String INPUT = "transform-input";

    /**
     * @return input channel.
     */
    @Input(TransformSink.INPUT)
    SubscribableChannel input();
}
```

TransformProcessor.java
```java
public interface TransformProcessor extends TransformSource, TransformSink {

}
```

MessageReceiver.java
```java
@Slf4j
@EnableBinding({Processor.class, TransformProcessor.class})
@Component
public class MessageReceiver {

    /**
     * 接收消息（与 transformMessage 接收的是同一个队列中的消息，一条消息只会被一个方法接收到
     *
     * @param message 消息
     */
    @StreamListener(Processor.INPUT)
    @SendTo(TransformProcessor.OUTPUT)
    public String handleMessage(String message) {
        log.info("Message received: " + message);
        return message.toLowerCase();
    }

    /**
     * 接收后转换消息（与 handleMessage 接收的是同一个队列中的消息，一条消息只会被一个方法接收到
     *
     * @param message 消息
     */
    @Transformer(inputChannel = Processor.INPUT, outputChannel = TransformProcessor.OUTPUT)
    public String transformMessage(String message) {
        log.info("Message received and transform: " + message);
        return message.toUpperCase();
    }

    /**
     * 接收转换后消息
     *
     * @param message 消息
     */
    @StreamListener(TransformProcessor.INPUT)
    public void handleTransformedMessage(String message) {
        log.info("Transformed Message received: " + message);
    }
}
```

application.yml
```yaml
server:
  port: 8081

spring:
  cloud:
    stream:
      bindings:
        input:
          # 对应 RabbitMQ 中的 Exchange（要与 output.destination 保持一致）
          destination: default-topic
          # 对应 RabbitMQ 中的 Queue
          group: default-topic-group
        transform-output:
          destination: transform-topic
          group: transform-topic-group
        transform-input:
          destination: transform-topic
          group: transform-topic-group
```

### 基于内容路由
#### 消费者
MessageReceiver.java
```java
@Slf4j
@EnableBinding({Sink.class})
@Component
public class MessageReceiver {

    /**
     * 接收消息
     *
     * @param message 消息
     */
    @StreamListener(target = Processor.INPUT, condition = "headers['version']=='1.5'")
    public void handleMessageWithHeader(String message) {
        log.info("Message received with header: " + message);
    }

    /**
     * 接收消息（不生效，原因还不清楚）
     *
     * @param message 消息
     */
    @StreamListener(target = Processor.INPUT, condition = "payload == 'Hello World, {2016}!'")
    public void handleMessageWithPayload(String message) {
        log.info("Message received with payload: " + message);
    }
}
```

#### 生产者
ProducerApplication.java
```java
@Slf4j
@SpringBootApplication
public class ProducerApplication implements CommandLineRunner {
    @Autowired
    private MessageSender messageSender;

    public static void main(String[] args) {
        SpringApplication.run(ProducerApplication.class, args);
    }

    @Override
    public void run(String... args) {
        for (int i = 0; i < 10; i++) {
            messageSender.sendMessage(MessageBuilder
                    .withPayload(String.format("Hello World, {201%s}!", i))
                    .setHeader("version", String.format("1.%s", i))
                    .build());
        }
    }
}
```

### 定时发送消息
#### 生产者
MessageSender.java
```java
@EnableBinding(Source.class)
@Component
public class MessageSender {

    /**
     * 定时发送消息：每隔 1s 发送一次，每次发送 1 条
     *
     * @return 消息
     */
    @Bean
    @InboundChannelAdapter(value = Source.OUTPUT, poller = @Poller(fixedDelay = "1000", maxMessagesPerPoll = "1"))
    public MessageSource<String> timerMessageSource() {
        return () -> new GenericMessage<>("Hello Spring Cloud Stream!");
    }
}
```

