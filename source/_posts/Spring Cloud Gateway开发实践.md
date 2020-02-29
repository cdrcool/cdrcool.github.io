---
title: Spring Cloud Gateway 开发实践
date: 2020-02-26 14:32:00
categories: Spring Cloud
tags:
    - Spring Spring Cloud Gateway
---
## Maven 依赖
```xml
<!-- 网关 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

## 路由转发
```yaml
spring:
  cloud:
    gateway:
      routes:
        # 秒杀服务
        - id: seckill
#          uri: lb://seckill
          uri: http://localhost:8081
          predicates:
            - Path=/api-seckill/**
        # 账号服务
        - id: account
#          uri: lb://account
          uri: http://localhost:8082
          predicates:
            - Path=/api-account/**
      default-filters:
        - StripPrefix=1
```

如上，我们访问网关地址 localhost:8080/api-account/ 时，经网关路由转发之后，地址就是 localhost:8082/。
* 如果我们要使用负载均衡，uri 需配置为 lb://account，account 为属性 spring.application.name 指定的名称。
* 示例中 StripPrefix=1 是必须的，表示 StripPrefix 过滤器将去掉 URL 路径中的第一个前缀，这里是 api-account。

## 熔断与降级
### Maven 依赖
```xml
<!-- 断路器 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

### 自动装配
可以通过设置 spring.cloud.circuitbreaker.resilience4j.enabled=false 来禁用 Resilience4J 自动装配。

### 配置
```yaml
spring:
  cloud:
    gateway:
      default-filters:
        # 断路器
        - name: CircuitBreaker
          args:
            name: circuitBreaker
            # 降级 URI
            fallback-uri: forward:/fallback
```

如上，我们访问网关地址 localhost:8080/api-seckill/ 时，如果此时 seckill 服务还未启动，那么网关会触发熔断，并将请求转发到 mapping 为 fallback 的 Controller，以实现服务降级。

在上面的示例中，我们是将请求转发到网关内部的 Controller，我们也可以将请求重新路由到外部应用程序中的 Controller，如下所示：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: ingredients
        uri: lb://ingredients
        predicates:
        - Path=//ingredients/**
        filters:
        - name: CircuitBreaker
          args:
            name: fetchIngredients
            fallbackUri: forward:/fallback
      - id: ingredients-fallback
        uri: http://localhost:9994
        predicates:
        - Path=/fallback
```

在新的示例中，当我们访问 ingredients 服务并出现熔断后，请求首先会被转发到 fallback，然后再被重新路由到 localhost:9994。

在前面的示例中，我们只是简单配置了 fallback，如果要对断路器做一些高级的配置，如熔断策略、异常处理等，参考 [Resilience4J 文档](https://cloud.spring.io/spring-cloud-circuitbreaker/reference/html/spring-cloud-circuitbreaker.html)

## 限流
### Maven 依赖
```xml
<!-- 限流（基于redis） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

### 配置
```yaml
spring:
  cloud:
    gateway:
      default-filters:
        # 限流
        - name: RequestRateLimiter
          args:
            # 限流匹配策略
            key-resolver: '#{@ipKeyResolver}'
            # 令牌桶的填充速率：用户每秒执行多少请求
            redis-rate-limiter.replenishRate: 10
            # 令牌桶的容量：用户在一秒钟内执行的最大请求数
            # 将此值设置为零将阻塞所有请求；将此值设置为高于 replenishRate，以允许临时突发
            redis-rate-limiter.burstCapacity: 20
```

如上：
* 限制了用户每秒最多执行 10 次请求；
* 限制了用户在一秒钟内最多执行 20 次请求；
* 限流匹配策略使用 ipKeyResolver(ip 限流)，ipKeyResolver 为 Spring Bean 的名称。

我们可以自定义各种 KeyResolver，如下所示：

```java
@Configuration
public class ThrottlingConfiguration {
    private static final String USER_ID_NAME = "userId";

    /**
     * 接口限流
     */
    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getPath().value());
    }

    /**
     * ip 限流
     */
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress());
    }

    /**
     * 用户限流
     */
    @Primary
    @Bean
    public KeyResolver principalNameKeyResolver() {
        return new PrincipalNameKeyResolver();
    }

    /**
     * 用户限流（要求请求路径中必须携带 userId 参数）
     */
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(Objects.requireNonNull(exchange.getRequest().getQueryParams().getFirst(USER_ID_NAME)));
    }
}
```

### 断路器配置
使用断路器，我们也能实现限流，示例：
```java
@Bean
    public Customizer<ReactiveResilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
                .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
                // 限制远程调用所花费的时间
                .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(5)).build()).build());
    }
```

## 请求耗时统计
```java
@Slf4j
@Component
public class ElapsedGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 请求执行之前的时间
        Long startTime = System.currentTimeMillis();
        return chain.filter(exchange).then().then(Mono.fromRunnable(() -> {
            // 请求执行之后的时间
            Long endTime = System.currentTimeMillis();
            log.info("请求 {} 耗时 {}ms", exchange.getRequest().getURI().getRawPath(), endTime - startTime);
        }));
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

如上，我们自定义了全局过滤器 ElapsedGlobalFilter，并将其注册为 Bean。因为该过滤器优先级最高，所以将其 order 设置为 Ordered.HIGHEST_PRECEDENCE。

