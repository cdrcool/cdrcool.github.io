---
title: Spring Cloud Circuit Breaker 开发实践
date: 2020-02-27 21:51:00
categories: Spring Cloud
---
### Maven 依赖
```xml
<!-- 断路器 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

### 启用/停用
可以通过设置 spring.cloud.circuitbreaker.resilience4j.enabled=false 来禁用 Resilience4J 自动装配。