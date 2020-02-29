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

## 查找服务
### 使用 Feign
**定义 API 接口：**
```java
@FeignClient(name = "account", path = "account")
public interface AccountApi {

    /**
     * 获取用户名
     *
     * @return 用户名
     */
    @GetMapping("username")
    String getUsername();
}
```

**调用 API 接口：**
```java
@Slf4j
@RequestMapping("seckill")
@RestController
public class SeckillController {
    @Autowired
    private AccountApi accountApi;

    @GetMapping("orderId")
    public String getOrderId() {
        log.info("Username: {}", accountApi.getUsername());
        return UUID.randomUUID().toString();
    }
}
```

### 使用 RestTemplate
**声明负载均衡 RestTemplate：**
```java
@LoadBalanced
@Bean
public RestTemplate loadbalancedRestTemplate() {
    return new RestTemplate();
}
```

**使用 RestTemplate：**
```java
@Slf4j
@RequestMapping("seckill")
@RestController
public class SeckillController {
    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("orderId")
    public String getOrderId() {
        log.info("Username: {}", restTemplate.getForObject("http://account/account/username", String.class));
        return UUID.randomUUID().toString();
    }
}
```

## 使用 DiscoveryClient
```java
@Autowired
private DiscoveryClient discoveryClient;

public String serviceUrl() {
    List<ServiceInstance> list = discoveryClient.getInstances("STORES");
    if (list != null && list.size() > 0 ) {
        URI uri = list.get(0).getUri();
        ...
    }
    return null;
}
```

## HTTP 健康检查
健康检查默认为 /actuator/health，时间间隔默认为 10s。我们可以通过以下属性进行修改：

```yaml
spring:
  cloud:
    consul:
      discovery:
        # 服务健康检查地址
        healthCheckPath: /actuator/health
        # 服务健康检查时间间隔
        healthCheckInterval: 10s
```

可以通过设置 `management.health.consun.enabled=false` 来禁用运行状况检查。