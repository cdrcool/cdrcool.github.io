---
title: Spring Boot--Spring Boot 应用监控 
date: 2021-06-16 10:56:16 
categories: Spring Boot
---

## 集成 Spring Boot Admin

### 创建 Spring Boot Admin Server

创建新工程，引入以下依赖：

```xml

<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
</dependency>
```

在应用启动类上添加注解`@EnableAdminServer`，代码如下：

```java

@EnableAdminServer
@SpringBootApplication
public class MonitorApplication {

    public static void main(String[] args) {
        SpringApplication.run(MonitorApplication.class, args);
    }
}
```

`application.yml` 配置如下：

```yaml
server:
  port: 8081

spring:
  application:
    name: op-monitor
```

### 创建 Spring Boot Admin Client

在 client 工程中引入以下依赖：

```xml

<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
</dependency>
```

在`application.yml` 中添加以下配置：

```yaml
spring:
  application:
    name: op-admin
  boot:
    admin:
      client:
        url: http://localhost:8081

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always
    logfile:
      enabled: true
```

### Spring Boot Admin 监控示例

启用 server 与 client 两个工程之后，访问 http://localhost:8081/，即可看到以下信息：

![Spring Boot Admin 示例1.png](/images/SpringBootAdmin示例1.png)
![Spring Boot Admin 示例2.png](/images/SpringBootAdmin示例2.png)
![Spring Boot Admin 示例3.png](/images/SpringBootAdmin示例3.png)