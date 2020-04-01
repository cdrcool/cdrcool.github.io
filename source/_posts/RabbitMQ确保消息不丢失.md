---
title: RabbitMQ 确保消息不丢失
date: 2020-01-28 14:49:00
categories: RabbitMQ
tags:
    - RabbitMQ
---
## 消息持久化
* Exchange 要持久化
```java
@Bean
public DirectExchange directExchange() {
    return new DirectExchange(DIRECT_EXCHANGE); // = new DirectExchange(DIRECT_EXCHANGE， true, false)
}
```

* Queue 要持久化
```java
@Bean
public Queue queue() {
    return new Queue(QUEUE); // = return new Queue(QUEUE, true, false)
}
```

* Message 要持久化
```java
 MessageProperties properties = new MessageProperties();
properties.setContentType("UTF-8");
properties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);

Message messageObj = new Message(message.getBytes(), properties);
rabbitTemplate.send(EXCHANGE, null, messageObj);
```

Exchange、Queue 和 Message 在默认情况下都是开启了持久化的。

## 消息确认
RabbitMQ 的消息确认有两种：
*  消息发送确认
用来确认生产者将消息发送给交换器，交换器传递给队列的过程中，消息是否成功投递。发送确认分为两步，一是确认是否到达交换器，二是确认是否到达队列。

* 消费接收确认
确认消费者是否成功消费了队列中的消息

### 消息发送确认
其实这个也不能叫确认机制，只是起到一个监听的作用，监听生产者是否发送消息到 Exchange 和 Queue。

#### 确认是否到达交换器
设置属性 publisher-confirm-type，并实现接口 RabbitTemplate.ConfirmCallback。

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: simple
```

```java
@Slf4j
@Component
public class RabbitTemplateConfig implements RabbitTemplate.ConfirmCallback {
    private final RabbitTemplate rabbitTemplate;

    public RabbitTemplateConfig(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    @PostConstruct
    public void init() {
        rabbitTemplate.setConfirmCallback(this);
    }

    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        log.info("correlationData: {}, ack: {}, cause: {}", correlationData, ack, cause);
    }
}
```

#### 确认是否到达队列
设置属性 publisher-returns，并实现 RabbitTemplate.ReturnCallback。

```yaml
spring:
  rabbitmq:
    # 确认消息发送到 Exchange 对应的 Queue
    publisher-returns: true
```

```java
@Slf4j
@Component
public class RabbitTemplateConfig implements RabbitTemplate.ReturnCallback {
    private final RabbitTemplate rabbitTemplate;

    public RabbitTemplateConfig(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    @PostConstruct
    public void init() {
        rabbitTemplate.setReturnCallback(this);
    }

    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("message: {}, replyCode: {}, replyText: {}, exchange: {}, routingKey: {}", message, replyCode, replyText, exchange, routingKey);
    }
}
```

注意，只有在消息从交换器发送到对应队列失败时才触发（比如根据发送消息时指定的 routingKey 找不到队列时会触发）。

### 消息接收确认
消费者确认分两种:自动确认和手动确认。默认为自动确认。
一旦设置成 manual 手动确认，一定要对消息做出应答，否则 RabbitMQ 会认为当前队列没有消费完成，将不再继续向该队列发送消息。

#### 自动确认
在自动确认模式中，消息在发送到消费者后即被认为"成功消费"。因此通常被称为“即发即忘”。
这种模式可以降低吞吐量（只要消费者可以跟上），以降低交付和消费者处理的安全性。

与手动确认模型不同，如果消费者的 TCP 连接或通道在真正的"成功消费"之前关闭，则服务器发送的消息将丢失。因此，自动消息确认应被视为不安全，并不适用于所有工作负载。

使用自动确认模式时需要考虑的另一件事是消费者过载。
手动确认模式通常与有界信道预取(BasicQos 方法)一起使用，该预取限制了信道上未完成（“进行中”）的消息的数量。
但是，自动确认没有这种限制。
因此，消费者可能会被消息的发送速度所淹没，可能会导致消息积压并耗尽堆或使操作系统终止其进程。某些客户端库将应用 TCP 反压(停止从套接字读取，直到未处理的交付积压超过某个限制)。
因此，仅建议能够以稳定的速度有效处理消息的消费者使用自动确认模式。

#### 手动确认
手动确认又分两种:肯定确认和否定确认。

要开启手动确认，需要设置 acknowledge-mode：
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        # 消费者手动确认消息
        acknowledge-mode: manual
```

* 肯定确认
```java
@RabbitListener(queues = QUEUE)
public void receive(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
    log.info("Receive message: {}", message);
    try {
        channel.basicAck(tag, false);
    } catch (IOException e) {
        log.error("Confirm message error", e);
    }
}
```

channel.basicAck(tag, false)：确认收到消息，消息将被队列移除，false 只确认当前 consumer 一个消息收到，true 确认所有 consumer 获得的消息。

* 否定确认
否定确认的场景不多，但有时候某个消费者因为某种原因无法立即处理某条消息时，就需要否定确认了。

否定确认时，需要指定是丢弃掉这条消息，还是让这条消息重新排队，过一会再来，又或者是让这条消息重新排队，并尽快让另一个消费者接收并处理它。

channel.basicNack(long, boolean,boolean)：确认否定消息，第一个 boolean 表示一个 consumer 还是所有，第二个 boolean 表示 requeue 是否重新回到队列，true 重新入队。
channel.basicReject(long, boolean)：拒绝消息，requeue=false 表示不再重新入队，如果配置了死信队列则进入死信队列。

当消息回滚到消息队列时，这条消息不会回到队列尾部，而是仍是在队列头部，这时消费者会又接收到这条消息，如果想消息进入队尾，须确认消息后再次发送消息。


参考资料：
[RabbitMQ的消息确认机制](https://www.cnblogs.com/fdzfd/p/9420333.html)
[RabbitMQ (十一) 消息确认机制 - 消费者确认](https://www.cnblogs.com/refuge/p/10356750.html)
[SpringBoot+RabbitMQ发送确认和消费手动确认机制](https://blog.csdn.net/yuyeqianhen/article/details/95065170)