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

## 消费消息
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

## 转换消息
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
### 消费消息
StreamApplication.java
```java
@Slf4j
@EnableBinding(value = {Processor.class})
@SpringBootApplication
public class StreamApplication implements CommandLineRunner {
    @Autowired
    private Processor processor;

    public static void main(String[] args) {
        SpringApplication.run(StreamApplication.class, args);
    }

    @StreamListener(Processor.INPUT)
    public void processMessage(String message) {
        log.info("Data received: " + message);
    }

    @Override
    public void run(String... args) {
        processor.output().send(MessageBuilder.withPayload("Hello World!").build());
    }
}
```

application.yml
```yaml
spring:
  cloud:
    stream:
      bindings:
        input:
          destination: default.messages
        output:
          destination: default.messages
```

应用程序启动之后，在控制台会看到以下输出：

```text
Data received: Hello World!
```

转到 RabbitMQ 管理控制台（localhost:15672/），并向 default.messages.anonymous.JdFLl8jMRxumpIUp6z_nXA 发送消息,在控制台也会看到对应的输出。

### 转换消息
StreamApplication.java
```java
@Slf4j
@SpringBootApplication
public class StreamApplication {

    public static void main(String[] args) {
        SpringApplication.run(StreamApplication.class, args);
    }
}
```

TimerSource.java
```java
@EnableBinding(Source.class)
public class TimerSource {

    @Bean
    @InboundChannelAdapter(value = Source.OUTPUT, poller = @Poller(fixedDelay = "10", maxMessagesPerPoll = "1"))
    public MessageSource<String> timerMessageSource() {
        return () -> new GenericMessage<>("Hello World!");
    }
}
```

TransformProcessor.java
```java
@Slf4j
@EnableBinding(Processor.class)
public class TransformProcessor {
    
    @Transformer(inputChannel = Processor.INPUT, outputChannel = Processor.OUTPUT)
    public Object transform(String message) {
        log.info("Data received: " + message);
        return message.toUpperCase();
    }
}
```

application.yml
```yaml
spring:
  cloud:
    stream:
      bindings:
        input:
          destination: default.messages
        output:
          destination: default.messages
```

应用程序启动之后，在控制台会看到以下输出：

```text
Data received: Hello World!
Data received: Hello World!
Data received: Hello World!
Data received: HELLO WORLD!
Data received: Hello World!
...
```

### 转发消息
```java
@Slf4j
@EnableBinding(value = {Processor.class})
@SpringBootApplication
public class StreamApplication implements CommandLineRunner {
    @Autowired
    private Processor processor;

    public static void main(String[] args) {
        SpringApplication.run(StreamApplication.class, args);
    }

    @Override
    public void run(String... args) {
        processor.output().send(MessageBuilder.withPayload("Hello World!").build());
    }
}
```

```java
@Slf4j
@EnableBinding(Processor.class)
public class TransformProcessor {

    @StreamListener(Processor.INPUT)
    @SendTo(Processor.OUTPUT)
    public String handle(String message) {
        log.info("Data received: " + message);
        return message.toUpperCase();
    }
}
```

应用程序启动之后，在控制台会看到以下输出：

```text
Data received: Hello World!
Data received: HELLO WORLD!
Data received: Hello World!
Data received: HELLO WORLD!
...
```