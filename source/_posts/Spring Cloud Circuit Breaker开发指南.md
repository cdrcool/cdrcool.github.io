---
title: Spring Cloud Circuit Breaker 开发指南
date: 2020-02-28 12:59:00
categories: Spring Cloud
---
Resilience4j 是一款轻量级，易于使用的容错库，其灵感来自于 Netflix Hystrix，专为 Java8 和函数式编程而设计。轻量级，因为库只使用了 Vavr，它没有任何其他外部依赖下。相比之下，Netflix Hystrix 对 Archaius 具有编译依赖性，Archaius 具有更多的外部库依赖性，例如 Guava 和 Apache Commons Configuration。

## Maven 依赖
```xml
<!-- 断路器 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

## 自动装配
可以通过设置 spring.cloud.circuitbreaker.resilience4j.enabled=false 来禁用 Resilience4J 自动装配。

## 断路器配置
### 配置属性
配置属性 | 默认值 | 描述
:-: | :-: | :-:
failureRateThreshold | 50（%） | 当故障率等于或大于阈值时，CircuitBreaker 转换为打开状态并开始短路调用。
slowCallRateThreshold | 100 | 配置百分比阈值。当慢速调用的百分比等于或大于阈值时，CircuitBreaker 转换为打开并开始短路调用。
slowCallDurationThreshold | 60（s） | 配置持续时间阈值，在该阈值以上，调用将被视为慢速并增加慢速调用的速率。
permittedNumberOfCallsInHalfOpenState | 10 | 配置当 CircuitBreaker 半开时允许的调用数。
slideWindowType | COUNT_BASED | 配置滑动窗口的类型，该窗口用于在 CircuitBreaker 关闭时记录调用结果。滑动窗口可以基于计数或基于时间。如果滑动窗口为 COUNT_BASED，slidingWindowSize 则记录并汇总最近的调用。 如果滑动窗口是 TIME_BASED，则 slidingWindowSize 记录并汇总最近几秒的调用。
slideWindowSize | 100 | 配置滑动窗口的大小，该窗口用于记录 CircuitBreaker 关闭时的调用结果。
minimumNumberOfCalls | 10 | 配置 CircuitBreaker 可以计算错误率之前所需的最小调用数（每个滑动窗口时段）。例如，如果 minimumNumberOfCalls 为 10，则在计算失败率之前，必须至少记录 10 个调用。如果仅记录了 9 个调用，则即使所有 9 个调用均失败，CircuitBreaker 也不会转换为打开。
waitDurationInOpenState | 60（s） | 从打开状态转为半开状态等待的时间。
recordExceptions | empty | 需要记录的异常列表。
ignoreExceptions | empty | 需要忽略的异常列表。
recordException | throwable -> true 默认情况下，所有异常都记录为失败。 | 用于评估是否应将异常记录为失败。如果异常应计为失败，则必须返回 true。如果异常应被视为成功，则必须返回 false，除非该异常被显式忽略 ignoreExceptions。
ignoreException | throwable ->false 默认情况下，不会忽略任何异常。 | 用于评估是否应忽略异常，并且该异常既不算作失败也不算成功。如果应忽略异常，则必须返回 true。否则必须返回 false。
automaticTransitionFromOpenToHalfOpenEnabled | false | 如果置为true，当等待时间结束会自动由打开变为半开，若置为 false，则需要一个请求进入来触发熔断器状态转换。

### 默认配置
要为所有断路器提供默认配置，需要创建一个自定义 bean，该 bean 通过一个 ReactiveResilience4JCircuitBreakerFactory 传递。可以使用 configureDefault 方法提供默认配置。

```java
@Bean
public Customizer<ReactiveResilience4JCircuitBreakerFactory> defaultCustomizer() {
    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
            // 限制远程调用所花费的时间
            .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(5)).build()).build());
}
```

### 特殊配置
与提供默认配置类似，我们可以创建一个自定义 bean，它传递给一个 ReactiveResilience4JCircuitBreakerFactory。

除了配置所创建的断路器之外，我们还可以在断路器创建之后，但在它返回给调用者之前，对其进行自定义。为此，可以使用 addCircuitBreakerCustomizer  方法。这对于向 Resilience4J 断路器添加事件处理程序是很有用的。

```java
@Bean
public Customizer<ReactiveResilience4JCircuitBreakerFactory> slowCusomtizer() {
    return factory -> {
        factory.configure(builder -> builder
        .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(5)).build())
        .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults()), "slow", "slowflux");

        factory.addCircuitBreakerCustomizer(circuitBreaker -> circuitBreaker.getEventPublisher()
            .onError(normalFluxErrorConsumer).onSuccess(normalFluxSuccessConsumer), "normalflux");
     };
}
```

### 收集指标
Spring Cloud Circuit Breaker Resilience4j 包含自动配置来设置指标集合，只要类路径上有正确的依赖项。要启用度量集合，需要引入依赖 org.springframework.boot:spring-boot-starter-actuator 和 io.github.resilience4j:resilience4j-micrometer。有关这些依赖项出现时生成的度量标准的更多信息，[参阅Resilience4j文档](https://resilience4j.readme.io/docs/micrometer)。

## Spring Retry 断路器配置
Spring Retry 为 Spring 应用程序提供声明性重试支持。项目的一部分包括实现断路器功能的能力。Spring Retry 通过它的 CircuitBreakerRetryPolicy 和有状态重试的组合提供了断路器实现。所有使用 Spring Retry 创建的断路器都将使用 CircuitBreakerRetryPolicy 和 DefaultRetryState 创建。这两个类都可以使用 SpringRetryConfigBuilder 来配置。

### 默认配置
要为所有断路器提供默认配置，需要创建一个通过 SpringRetryCircuitBreakerFactory 传递的定制 bean。可以使用 configureDefault 方法提供默认配置。

```java
@Bean
public Customizer<SpringRetryCircuitBreakerFactory> defaultCustomizer() {
    return factory -> factory.configureDefault(id -> new SpringRetryConfigBuilder(id)
        .retryPolicy(new TimeoutRetryPolicy()).build());
}
```

### 特殊配置
与提供默认配置类似，我们可以创建一个自定义 bean，它通过 SpringRetryCircuitBreakerFactory 传递。

```java
@Bean
public Customizer<SpringRetryCircuitBreakerFactory> slowCustomizer() {
    return factory -> factory.configure(builder -> builder.retryPolicy(new SimpleRetryPolicy(1)).build(), "slow");
}
```

除了配置所创建的断路器之外，我们还可以在断路器创建之后，但在它返回给调用者之前，对其进行自定义。为此，可以使用 addRetryTemplateCustomizers 方法。这对于将事件处理程序添加到 RetryTemplate 非常有用。

```java
@Bean
public Customizer<SpringRetryCircuitBreakerFactory> slowCustomizer() {
    return factory -> factory.addRetryTemplateCustomizers(retryTemplate -> retryTemplate.registerListener(new RetryListener() {

        @Override
        public <T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback) {
            return false;
        }

        @Override
        public <T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {

        }

        @Override
        public <T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {

        }
    }));
}
```

## 参考资料
1. [Spring Cloud Circuit Breaker](https://www.liangzl.com/get-article-detail-154284.html)