---
title: Spring Cloud Consul 开发实践
date: 2020-02-29 11:13:00
categories: Spring Cloud
tags:
    - Spring Cloud Consul
    - Consul
---
## 服务发现
### Maven 依赖
```xml
<!-- 服务注册与发现 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

### 配置属性
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
        # 连接不上 Consul，则抛出异常，终止应用程序启动
        # 本地开发时，可能需要设置为 false
        failFast: true
```

* 在上例中，consul 相关配置都是使用的默认值，即不配置也可以运行。
* 由于 Spring Cloud 默认会采用 Ribbon 作为负载均衡实现，而这里没有使用 Eureka，所以需要禁用到 Ribbon 负载均衡。
* 如果使用了 Spring Cloud Config，属性 spring.application.name 需配置在 bootstrap.yml 中。
* 如果使用了 Spring Cloud Consul Config，consul 相关属性需配置在 bootstrap.yml 中。

### 查找服务
#### 使用 Feign
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

#### 使用 RestTemplate
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

### 使用 DiscoveryClient
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

### HTTP 健康检查
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

## 配置中心
### Maven 依赖
```xml
<!-- 服务注册与发现 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

### 配置属性
```yaml
spring:
  application:
    name: account
  cloud:
    consul:
      # 启用 consul
      enabled: true
      # 集群地址
      host: localhost
      # 集群端口
      port: 8500
      # 配置中心
      config:
        # 启用配置中心
        enabled: true
        # 根文件夹
        prefix: config
        # 文件格式，默认值 KEY_VALUE
        format: YAML
        # 文件名
        data-key: data
        # 应用名和环境分隔符，默认值 ,
        profile-separator: '-'
        # 连接不上 Consul，则抛出异常，终止应用程序启动
        # 本地开发时，可能需要设置为 false
        failFast: true
```

* 配置属性需放在 bootstrap.yml 文件中。
* 在上例中，除 format 和 profile-separator 属性外，其它 consul 相关配置都是使用的默认值，即不配置也可以运行。

### 创建配置文件
1. 在 Consul 界面 Key/Value 菜单下创建目录：
* config/account/data account 应用的默认配置
* config/account-dev/data account 应用的 dev 环境配置

我们也可以创建另外两个目录：
* config/application/data 所有应用的默认配置
* config/application-dev/data 所有应用的 dev 环境配置

2. 在 data 路径下添加配置属性：
```yaml
server:
  port: 8082

spring:
  application:
    name: account
  cloud:
    loadbalancer:
      ribbon:
        # 由于未使用 Eureka，需要禁用掉默认的 Ribbon 负载均衡
        enabled: false
    consul:
      # 启用 consul
      enabled: true
      # 集群地址
      host: localhost
      # 集群端口
      port: 8500
      # 服务发现
      discovery:
        # 启用服务发现
        enabled: true
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
        # 连接不上 Consul，则抛出异常，终止应用程序启动
        # 本地开发时，可能需要设置为 false
        failFast: true
      watch:
        # 启动 watch
        enabled: true
        # watch 调用频率，单位：毫秒
        delay: 1000
        # watch 查询阻塞时间，单位：秒
        # 默认值 55 秒，这意味着连续两次快速的修改配置属性，属性不会立即更新为最新的值，因为前面的 watch 查询阻塞了后面的查询
        wait-time: 55

management:
  endpoints:
    web:
      # 默认只暴露了 health/info 端点
      exposure:
        include: health,info,metrics,refresh,gateway
```

经过以上两步操作之后，在启动应用程序之后，便会从 Consul 中加载配置属性。

### 动态刷新配置属性
如果 Consul 中配置属性发生了变化，我们可以通过向 /refresh 端点发送 HTTP POST 请求，来重新加载配置。 /refresh 端点会将发生变化（新增、修改或删除）的属性列表返回给我们。

但是， /refresh 端点并不能动态刷新应用程序中的属性，要做到这点，还需在用到属性的 bean 上添加 @RefreshScope 注解，如下所示：

```java
@Slf4j
@RefreshScope
@RequestMapping("account")
@RestController
public class AccountController {
    @Value("${username:admin}")
    private String username;

    @GetMapping("username")
    public String getUsername() {
        return username;
    }
}
```

其实，即使我们不调用 /refresh 端点，配置属性也会自动刷新，这是因为 Consul Config Watch 利用 Consul 的能力来监控配置属性是否发生变化。我们可以通过设置 watch 相关属性，来启用或关闭 watch，及调整 watch 频率等，如下所示：

```yaml
spring:
  cloud:
    consul:
      watch:
        # 启动 watch
        enabled: true
        # watch 调用频率，单位：毫秒
        delay: 1000
        # watch 查询阻塞时间，单位：秒
        # 默认值 55 秒，这意味着连续两次快速的修改配置属性，属性不会立即更新为最新的值，因为前面的 watch 查询阻塞了后面的查询
        wait-time: 55
```

## Consul 重试
在应用程序启动时，如果遇到 Consul Agent 连接不上，可以设置重试次数。只需引入以下 2 个依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.retry</groupId>
        <artifactId>spring-retry</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

默认情况下，会重试 6 次，初始回退间隔为 1000 ms，后续回退的指数乘数为 1.1。我们可以使用 spring.cloud.consul.retry.* 配置属性来配置这些属性(以及其它属性)。

Consul 重试既可以在服务发现中使用，也可以在配置中心中使用。