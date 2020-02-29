---
title: Spring Cloud OpenFeign 开发实践
date: 2020-02-29 10:03:00
categories: Spring Cloud
tags:
    - Spring Cloud OpenFeign
    - Spring Cloud LoadBalancer
    - Ribbon
---
## Maven 依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

## 基本使用
1. 在 account-api 模块中定义 API 接口 AccountApi。

```java
@FeignClient(name = "account", url = "http://localhost:8082", path = "account")
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

* name 客户端名称，必要属性。如果项目使用了服务发现，name 属性需指定为微服务的名称。
* url 服务地址，一般用于调试。
* path 请求的 mapping 前缀。
* fallback 定义容错的处理类，当调用远程接口失败或超时时，会调用对应接口的容错逻辑，fallback 指定的类必须实现 @FeignClient 标记的接口。
* fallbackFactory 工厂类，用于生成 fallback 类实例，通过这个属性我们可以实现每个接口通用的容错逻辑，减少重复的代码。
* configuration Feign 配置类，可以自定义 Feign 的 Encoder、Decoder、Contract、LogLevel。
* decode404 当发生 HTTP 404 错误时，如果该属性为 true，会调用 decoder 进行解码，否则抛出 FeignException。

2. 在 account-api 模块中定义 AccountController。

```java
@Slf4j
@RequestMapping("account")
@RestController
public class AccountController {

    @GetMapping("username")
    public String getUsername() {
        return "admin";
    }
}
```

**注意：AccountController 不应该继承 AccountApi，因为这会导致服务的调用方和提供方紧密耦合。同时在 SpringMVC 中会有问题，因为请求参数映射是不能被继承的。** 

3. 在 seckill-web 中启动 Feign 客户端。

```java
@EnableFeignClients(basePackages = {"account"})
@SpringBootApplication
public class SeckillApplication {

    public static void main(String[] args) {
        SpringApplication.run(SeckillApplication.class, args);
    }

}
```

**注意：** 默认会扫描启动类所在根路径下的所有使用了注解 @FeignClient 的 Feign 客户端，如果 Feign 客户端不在启动类所在根路径下，则需要设置 @EnableFeignClients 的属性 basePackages 来指定要扫描的包。

5. 在 seckill-web 中使用 API 接口。
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