---
title: Spring Cloud Consul 开发实践
date: 2020-02-29 11:13:00
categories: Spring Cloud
tags:
    - Spring Cloud Consul
---
## Maven 依赖
```xml
<!-- 服务注册与发现 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

## 配置属性
```yaml
spring:
  application:
    name: account
  cloud:
    loadbalancer:
      ribbon:
        # 由于未使用 Eureka，需要禁用掉默认的 Ribbon 负载均衡
        enabled: false
    consul:
      # 启用服务发现
      enabled: true
      # 集群地址
      host: localhost
      # 集群端口
      port: 8500
      discovery:
        # 启用服务注册
        register: true
        # 服务注册名称
        serviceName: ${spring.application.name}
        # 服务注册 ID
        instanceId: ${spring.application.name}:${spring.cloud.consul.port}
        # 服务健康检查地址
        healthCheckPath: /actuator/health
        # 服务健康检查时间间隔
        healthCheckInterval: 10s
```

* 在上例中，consul 相关配置都是使用的默认值，即不配置也可以运行。
* 由于 Spring Cloud 默认会采用 Ribbon 作为负载均衡实现，而这里没有使用 Eureka，所以需要禁用到 Ribbon 负载均衡。
* 如果使用了 Spring Cloud Config，属性 spring.application.name 需配置在 bootstrap.yml 中。
* 如果使用了 Spring Cloud Consul Config，consul 相关属性需配置在 bootstrap.yml 中。

## 服务调用