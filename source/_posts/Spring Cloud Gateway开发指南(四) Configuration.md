---
title: Spring Cloud Gateway 开发指南（四） Configuration
date: 2020-02-27 20:35:00
categories: Spring Cloud
tags:
    - Spring Cloud Gateway
---
## 配置
Spring Cloud Gateway 的配置由 RouteDefinitionLocator 实例集合驱动。以下是 RouteDefinitionLocator 的接口定义：

```java
public interface RouteDefinitionLocator {

	Flux<RouteDefinition> getRouteDefinitions();

}
```

默认情况下，PropertiesRouteDefinitionLocator 通过使用 Spring Boot 的 @ConfigurationProperties 机制加载属性。

有两种方式可以配置：使用命名参数和使用带有位置参数的快捷符号。下面两个例子是等价的：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setstatus_route
        uri: https://example.org
        filters:
        - name: SetStatus
          args:
            status: 401
      - id: setstatusshortcut_route
        uri: https://example.org
        filters:
        - SetStatus=401
```
对于网关的某些用法，属性配置是足够的，但是一些生产用例可以从外部源（如数据库）加载配置中获益。将来的里程碑版本将有基于 Spring 数据存储库（如Redis、MongoDB和Cassandra）的 RouteDefinitionLocator 实现。

## 路由元数据配置
可以使用元数据为每个路由配置额外的参数，如下所示：

```yaml
pring:
  cloud:
    gateway:
      routes:
      - id: route_with_metadata
        uri: https://example.org
        metadata:
          optionName: "OptionValue"
          compositeObject:
            name: "value"
          iAmNumber: 1
```

可以从 exchange 中获取所有元数据属性，如下所示：

```java
Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
// get all metadata properties
route.getMetadata();
// get a single metadata property
route.getMetadata(someKey);
```

## HTTP 超时配置
可以为所有路由配置 Http 超时(响应和连接)，并覆盖每个特定路由。

### 全局超时
配置全局 Http 超时：
* 必须以毫秒为单位指定连接超时
* 响应超时必须指定为 java.time.Duration

全局 Http 超时示例：
```yaml
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 1000
        response-timeout: 5s
```

### 单个路由超时
配置单个路由超时：
* 必须以毫秒为单位指定连接超时
* 响应超时必须以毫秒为单位指定

单个路由的 Http 超时示例-配置文件：
```yaml
- id: per_route_timeouts
  uri: https://example.org
  predicates:
    - name: Path
      args:
        pattern: /delay/{timeout}
  metadata:
    response-timeout: 200
    connect-timeout: 200
```

单个路由的 Http 超时示例-Java DSL：
```java
import static org.springframework.cloud.gateway.support.RouteMetadataUtils.CONNECT_TIMEOUT_ATTR;
import static org.springframework.cloud.gateway.support.RouteMetadataUtils.RESPONSE_TIMEOUT_ATTR;

@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder routeBuilder){
 return routeBuilder.routes()
       .route("test1", r -> {
          return r.host("*.somehost.org").and().path("/somepath")
                .filters(f -> f.addRequestHeader("header1", "header-value-1"))
                .uri("http://someuri")
                .metadata(RESPONSE_TIMEOUT_ATTR, 200)
                .metadata(CONNECT_TIMEOUT_ATTR, 200);
       })
       .build();
}
```

### Fluent Java 路由 API
为了允许在 Java 中进行简单配置，RouteLocatorBuilder bean 提供了 Fluent API。示例：

```java
// static imports from GatewayFilters and RoutePredicates
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder, ThrottleGatewayFilterFactory throttle) {
    return builder.routes()
            .route(r -> r.host("**.abc.org").and().path("/image/png")
                .filters(f ->
                        f.addResponseHeader("X-TestHeader", "foobar"))
                .uri("http://httpbin.org:80")
            )
            .route(r -> r.path("/image/webp")
                .filters(f ->
                        f.addResponseHeader("X-AnotherHeader", "baz"))
                .uri("http://httpbin.org:80")
                .metadata("key", "value")
            )
            .route(r -> r.order(-1)
                .host("**.throttle.org").and().path("/get")
                .filters(f -> f.filter(throttle.apply(1,
                        1,
                        10,
                        TimeUnit.SECONDS)))
                .uri("http://httpbin.org:80")
                .metadata("key", "value")
            )
            .build();
}
```

这种风格还允许使用更多的定制断言。RouteDefinitionLocator bean 定义的谓词列表使用逻辑 AND。通过使用 Fluent Java API，我们可以在类 Predicate 上使用 and()、or() 和 negate() 操作符。

### DiscoveryClient 路由定义 Locator
可以使用与 DiscoveryClient 兼容的服务注册中心里注册的服务来创建路由。

要启用此功能，需要设置 spring.cloud.gateway.discovery.locator.enabled=true，并确保类路径上有一个 DiscoveryClient(如 Netflix Eureka、或Zookeeper) 实现。

### 为 DiscoveryClient 路由配置谓词和过滤器
默认情况下，网关为使用 DiscoveryClient 创建的路由定义单个谓词和过滤器。

默认谓词是使用模式 /serviceId/** 定义的路径谓词，其中 serviceId 是来自 DiscoveryClient 的服务的 ID。

默认的过滤器是一个重写路径过滤器，它使用正则表达式 /serviceId/(?<remaining>.*) 和替换的 /${remaining}。这将在请求向下游发送之前从路径中删除服务 ID。

如果希望自定义 DiscoveryClient 路由使用的谓词或过滤器，需设置 spring.cloud.gateway.discovery.locator.predicates[x] 和 spring.cloud.gateway.discovery.locator.filters[y]。这样做时，如果希望保留该功能，则需要确保包含前面显示的缺省谓词和筛选器。示例：

```yaml
spring.cloud.gateway.discovery.locator.predicates[0].name: Path
spring.cloud.gateway.discovery.locator.predicates[0].args[pattern]: "'/'+serviceId+'/**'"
spring.cloud.gateway.discovery.locator.predicates[1].name: Host
spring.cloud.gateway.discovery.locator.predicates[1].args[pattern]: "'**.foo.com'"
spring.cloud.gateway.discovery.locator.filters[0].name: Hystrix
spring.cloud.gateway.discovery.locator.filters[0].args[name]: serviceId
spring.cloud.gateway.discovery.locator.filters[1].name: RewritePath
spring.cloud.gateway.discovery.locator.filters[1].args[regexp]: "'/' + serviceId + '/(?<remaining>.*)'"
spring.cloud.gateway.discovery.locator.filters[1].args[replacement]: "'/${remaining}'"
```

## CORS 配置
可以配置网关来控制 CORS 行为。全局 CORS 配置是 Spring Framework CorsConfiguration 的 URL 模式映射。示例：

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "https://docs.spring.io"
            allowedMethods:
            - GET
```

在前面的示例中，CORS 请求允许来自 docs.spring.io 的所有 GET 请求路径。

要为某些网关路由谓词不能处理的请求提供相同的 CORS 配置，需设置 spring.cloud.gateway.globalcors.add-to-simple-url-handler-mapping=true。当我们试图支持 CORS preflight 请求时，这是很有用的，因为 HTTP 方法是 options，所以路由谓词不为 true。

## 日志级别
下面的 loggers 可能包含在 DEBUG 和 TRACE 级别上有价值的故障排除信息。

* org.springframework.cloud.gateway
* org.springframework.http.server.reactive
* org.springframework.web.reactive
* org.springframework.boot.autoconfigure.web
* reactor.netty
* redisratelimiter

## TLS 和 SSL
网关可以通过 Spring server 配置来监听 HTTPS 请求。示例：

```yaml
server:
  ssl:
    enabled: true
    key-alias: scg
    key-store-password: scg1234
    key-store: classpath:scg-keystore.p12
    key-store-type: PKCS12
```

可以将网关路由路由到 HTTP 和 HTTPS 后端。如果是路由到一个 HTTPS 后端，可以使用下面的配置使得网关信任所有下游证书：

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          useInsecureTrustManager: true
```

使用不安全的信任管理器不适合生产环境。对于产品部署，可以使用一组已知的证书配置网关，网关可以通过以下配置信任这些证书：

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          trustedX509Certificates:
          - cert1.pem
          - cert2.pem
```

如果 Spring Cloud Gateway 没有提供受信任的证书，则使用默认的信任存储区（可以通过设置 javax.net.ssl.trustStore 系统属性来覆盖该存储区）。

### TLS 握手
网关维护一个用于路由到后端的客户端池。当通过 HTTPS 通信时，客户端发起 TLS 握手。许多超时都与此握手相关。可以使用下面的配置来调整超时配置：

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          handshake-timeout-millis: 10000
          close-notify-flush-timeout-millis: 3000
          close-notify-read-timeout-millis: 0
```

## 参考资料
1. [Spring Cloud Gateway](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.1.RELEASE/reference/html/)