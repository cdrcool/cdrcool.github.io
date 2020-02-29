---
title: Spring Cloud Gateway 开发指南（二） GatewayFilter Factories
date: 2020-02-27 11:33:00
categories: Spring Cloud
tags:
    - Spring Spring Cloud Gateway
---
路由过滤器允许以某种方式修改传入的 HTTP 请求或返回的 HTTP 响应。路由过滤器的作用域是特定的路由。Spring Cloud Gateway 包含许多内置的 GatewayFilter Factories。

## AddRequestHeader
接受名称和值两个参数。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        filters:
        - AddRequestHeader=X-Request-red, blue
```

这将为所有匹配的请求在下游请求的 headers 中添加 header X-Request-red:blue。

AddRequestHeader 知道用于匹配路径或主机的 URI 变量。URI 变量可以在值中使用，并在运行时展开。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment}
        filters:
        - AddRequestHeader=X-Request-Red, Blue-{segment}
```

## AddRequestParameter
接受名称和值两个参数。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_parameter_route
        uri: https://example.org
        filters:
        - AddRequestParameter=red, blue
```

这将为所有匹配的请求在下游请求的查询字符串中添加 red=blue。

AddRequestParameter 知道用于匹配路径或主机的 URI 变量。URI 变量可以在值中使用，并在运行时展开。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_parameter_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - AddRequestParameter=foo, bar-{segment}
```

## AddResponseHeader
接受名称和值两个参数。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_response_header_route
        uri: https://example.org
        filters:
        - AddResponseHeader=X-Response-Red, Blue
```

这将为所有匹配的请求在下游响应的 headers 中添加 header X-Response-Red:Blue。

AddResponseHeader 知道用于匹配路径或主机的 URI 变量。URI 变量可以在值中使用，并在运行时展开。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_response_header_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - AddResponseHeader=foo, bar-{segment}
```

## DedupeResponseHeader
接受名称参数和可选策略参数。名称可以包含以空格分隔的 header 名列表。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: dedupe_response_header_route
        uri: https://example.org
        filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
```

当网关 CORS 逻辑和下游逻辑都添加它们时，这将从相应的 headers 中移除重复的 Access-Control-Allow-Credentials 和 Access-Control-Allow-Origin。

DedupeResponseHeader 过滤器也接受一个可选的策略参数。接受的值是 RETAIN_FIRST(默认值)、RETAIN_LAST 和 RETAIN_UNIQUE。

## Spring Cloud CircuitBreaker
此过滤器使用 Spring Cloud CircuitBreaker API 将 Gateway 路由封装在断路器中。Spring Cloud CircuitBreaker 支持两个可与 Spring Cloud Gateway 一起使用的库，即 Hystrix 和 Resilience4J。由于 Netflix 将 Hystrix 置于仅维护模式，建议大家使用 Resilience4J。

要启用此 filter，需要将 spring-cloud-starter-circuitbreaker-reactor-resilience4j 或 spring-cloud-starter-netflix-hystrix 置于类路径中。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: circuitbreaker_route
        uri: https://example.org
        filters:
        - CircuitBreaker=myCircuitBreaker
```

要配置断路器，请参阅所使用的断路器实现的配置。
* [Resilience4J 文档](https://cloud.spring.io/spring-cloud-circuitbreaker/reference/html/spring-cloud-circuitbreaker.html)
* [Hystrix 文档](https://cloud.spring.io/spring-cloud-netflix/reference/html/)

此过滤器也可以接受一个可选的 fallbackUri 参数。目前仅支持转发：支持 schemed URIs。如果调用了 fallback，请求将被转发给 URI 匹配的 controller。下面的示例配置了这样的 fallback：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: circuitbreaker_route
        uri: lb://backing-service:8088
        predicates:
        - Path=/consumingServiceEndpoint
        filters:
        - name: CircuitBreaker
          args:
            name: myCircuitBreaker
            fallbackUri: forward:/inCaseOfFailureUseThis
        - RewritePath=/consumingServiceEndpoint, /backingServiceEndpoint
```

下面的代码在 Java 中做了相同的事情：
```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("circuitbreaker_route", r -> r.path("/consumingServiceEndpoint")
            .filters(f -> f.circuitBreaker(c -> c.name("myCircuitBreaker").fallbackUri("forward:/inCaseOfFailureUseThis"))
                .rewritePath("/consumingServiceEndpoint", "/backingServiceEndpoint")).uri("lb://backing-service:8088")
        .build();
}
```

此示例在调用断路器 fallback 时转发到 /inCaseofFailureUseThis URI。注意，这个示例还演示了 Spring Cloud Netflix Ribbon 负载平衡(可选)(由目标 URI 上的 lb 前缀定义)。

主要的场景是使用 fallbackUri 来定义网关应用程序中的内部控制器或处理程序。但是，我们也可以将请求重新路由到外部应用程序中的控制器或处理程序，如下所示：

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

在本例中，网关应用程序中没有 fallback endpoint 或 handler。但是，在另一个应用程序中有一个，注册在 localhost:9994 下。

在请求被转发到 fallback 的情况下，此过滤器也提供了导致 fallback 的 Throwable。它作为属性 ServerWebExchangeUtils.CIRCUITBREAKER_EXECUTION_EXCEPTION_ATTR 被添加到 ServerWebExchange 中，可以在处理网关应用程序中的 fallback 时使用。

## FallbackHeaders
此过滤器允许向转发到外部应用程序的 fallbackUri 的请求的 headers 中添加  Hystrix 或 Spring Cloud CircuitBreaker 执行异常细节。示例：

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
        filters:
        - name: FallbackHeaders
          args:
            executionExceptionTypeHeaderName: Test-Header
```

在本例中，在运行断路器时发生执行异常后，请求被转发到运行在 localhost:9994 上的应用程序中的 fallback endpoint 或 handler。包含异常类型、消息和根原因异常类型和消息的 headers 由此过滤器添加到该请求。

可以通过设置以下参数的值(与默认值一起显示)来覆盖配置中 header 的名称:
* executionExceptionTypeHeaderName (“Execution-Exception-Type”)
* executionExceptionMessageHeaderName (“Execution-Exception-Message”)
* rootCauseExceptionTypeHeaderName (“Root-Cause-Exception-Type”)
* rootCauseExceptionMessageHeaderName (“Root-Cause-Exception-Message”)

## MapRequestHeader
它创建了一个新的 header(toHeader)，其值是从传入的 http 请求的 header(fromHeader)中提取出来的。如果传入的 header 不存在，则过滤器没有影响。如果新的 header 已经存在，它的值将被新值扩充。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: map_request_header_route
        uri: https://example.org
        filters:
        - MapRequestHeader=Blue, X-Request-Red
```

将 X-Request-Red:<values> header 添加到下游请求中，并将其值更新为传入的 HTTP 请求中的 Blue header 的值。

## PrefixPath
接受单个 prefix 参数。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - PrefixPath=/mypath
```

这将为所有匹配请求的路径加上前缀 /mypath。因此，对 /hello 的请求将被发送到 /mypath/hello。

## PreserveHostHeader
没有参数。此过滤器设置路由过滤器检查的请求属性，以确定是否应该发送原始 host header，而不是由 HTTP 客户端确定的 host header。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: preserve_host_route
        uri: https://example.org
        filters:
        - PreserveHostHeader
```

## RequestRateLimiter
此过滤器使用一个 RateLimiter 实现来确定当前请求是否被允许继续。如果不是，则返回 HTTP 429 的状态—太多的请求(默认情况下)。

此过滤器接受一个可选的 keyResolver 参数和特定于速率限制器的参数。

keyResolver 是实现 keyResolver 接口的 bean。在配置中，使用 SpEL 按名称引用 bean。#{@myKeyResolver} 是一个 SpEL 表达式，它引用一个名为 myKeyResolver 的 bean。下面的代码显示了 KeyResolver 接口：

```java
public interface KeyResolver {
    Mono<String> resolve(ServerWebExchange exchange);
}
```

KeyResolver 的默认实现是 PrincipalNameKeyResolver，它从 ServerWebExchange 中获取 Principal  并调用 Principal.getname()。

默认情况下，如果 KeyResolver 没有找到 key，请求将被拒绝。可以通过设置 spring.cloud.gateway.filter.request-rate-limiter.deny-empty-key (true or false) 和 spring.cloud.gateway.filter.request-rate-limiter.empty-key-status-code 属性来调整此行为。

### Redis RateLimiter
Redis 的实现基于 [Stripe](https://stripe.com/blog/rate-limiters) 完成的工作。它需要使用 spring-boot-starter-data-redis-reactive Spring Boot starter。

使用的算法是[令牌桶算法（Token Bucket Algorithm）](https://en.wikipedia.org/wiki/Token_bucket)。

* redis-rate-limiter.replenishRate 允许用户每秒执行多少请求，而不需要删除任何请求。这是令牌桶被填充的速率。

* redis-rate-limiter.burstCapacity 用户在一秒钟内允许执行的最大请求数。这是令牌桶可以容纳的令牌数量。将此值设置为 0 将阻塞所有请求。

一个稳定的速率是通过设置相同的 replenishRate 和 burstCapacity 来实现的。可以通过将 burstCapacity 设置为高于 replenishRate 来允许临时突发。在这种情况下，需要允许速率限制器在突发之间留出一些时间(根据 replenishRate)，因为两个连续的突发将导致请求下降(HTTP 429—请求太多)。

示例：
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: requestratelimiter_route
        uri: https://example.org
        filters:
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
```

下面的示例在 Java 中配置一个 KeyResolver：

```java
@Bean
KeyResolver userKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));
}
```

这定义了每个用户的请求速率限制为 10。允许 20 个请求，但是在接下来的一秒内，只有 10 个请求可用。KeyResolver 是一个简单的工具，它获取用户请求参数(注意，不建议在生产环境中使用这个参数)。

我们还可以将速率限制器定义为实现了 RateLimiter 接口的 bean。在配置中，可以使用 SpEL 通过名称引用 bean。#{@myRateLimiter} 是一个 SpEL 表达式，它引用一个名为 myRateLimiter 的 bean。下面的配置定义了一个速率限制器，它使用前面配置中定义的 KeyResolver：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: requestratelimiter_route
        uri: https://example.org
        filters:
        - name: RequestRateLimiter
          args:
            rate-limiter: "#{@myRateLimiter}"
            key-resolver: "#{@userKeyResolver}"
```

## RedirectTo
接受两个参数：status 和 url。状态参数应该是 300 系列的重定向 HTTP 代码，比如 301。url 参数应该是一个有效的 url。这是 Location header 的值。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: prefixpath_route
        uri: https://example.org
        filters:
        - RedirectTo=302, https://acme.org
```

这将发送一个 302 状态码来执行重定向，重定向的地址为 https://acme.org。

## RemoveHopByHopHeadersFilter
从转发的请求中移除 headers。

默认移除的 headers 由：
* Connection
* Keep-Alive
* Proxy-Authenticate
* Proxy-Authorization
* TE
* Trailer
* Transfer-Encoding
* Upgrade

要更改移除的 headers，可以设置 spring.cloud.gateway.filter.remove-non-proxy-header.headers 属性来指明要删除的 headers 列表。

## RemoveRequestHeader
接受名称参数。它是要删除的 header 的名称。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: removerequestheader_route
        uri: https://example.org
        filters:
        - RemoveRequestHeader=X-Request-Foo
```

这将在 X-Request-Foo header 被发送到下游之前删除它。

## RemoveResponseHeader
接受名称参数。它是要删除的 header 的名称。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: removeresponseheader_route
        uri: https://example.org
        filters:
        - RemoveResponseHeader=X-Response-Foo
```

这将在响应返回到网关客户端之前从响应中删除 X-Response-Foo header。

要删除任何类型的敏感标头，我们应该为希望删除的任何 routes 配置这个 filter。此外，还可以使用 spring.cloud.gateway.default-filters 配置此过滤器一次，并将其应用于所有 routes。

## RemoveRequestParameter
接受名称参数。它是要删除的查询参数的名称。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: removerequestparameter_route
        uri: https://example.org
        filters:
        - RemoveRequestParameter=red
```

这将在 red 参数被发送到下游之前删除它。

## RewritePath
接受一个路径 regexp 参数和一个替换参数。它使用 Java 正则表达式以一种灵活的方式重写请求路径。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://example.org
        predicates:
        - Path=/foo/**
        filters:
        - RewritePath=/red(?<segment>/?.*), $\{segment}
```

对于 /red/blue 的请求路径，这将在发出下游请求之前将路径设置为 /blue。注意，由于 YAML 规范的原因，应该将 $ 替换为 $\。

## RewriteLocationResponseHeader
修改 Location 响应 header 的值，通常去掉后端特定的细节。它接受 stripVersionMode、locationHeaderName、hostValue 和 protocolsRegex 参数。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: rewritelocationresponseheader_route
        uri: http://example.org
        filters:
        - RewriteLocationResponseHeader=AS_IN_REQUEST, Location, ,
```

例如，对于请求 POST api.example.com/some/object/name，object-service.prod.example.net/v2/some/object/id 的 Location 响应 header 被重写为 api.example.com/some/object/id。

stripVersionMode 参数有以下可能的值：NEVER_STRIP、AS_IN_REQUEST(默认值) 和 ALWAYS_STRIP。
* NEVER_STRIP 即使原始请求路径不包含 version，也不会删除 version。
* AS_IN_REQUEST 仅当原始请求路径不包含 version 时，才剥离 version。
* ALWAYS_STRIP 即使原始请求路径包含 version，version 也总是被剥离。

如果提供了 hostValue 参数，则该参数用于替换响应的 Location header 的 host:port 部分。如果没有提供，则使用 Host request header。

protocolsRegex 参数必须是有效的 regex 字符串，协议名与之匹配。如果不匹配，则筛选器什么也不做。默认是 http|https|ftp|ftps。

## RewriteResponseHeader
接受名称、regexp 和替换参数。它使用 Java 正则表达式以一种灵活的方式重写响应头值。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: rewriteresponseheader_route
        uri: https://example.org
        filters:
        - RewriteResponseHeader=X-Response-Red, , password=[^&]+, password=***
```

对于 header 值 /42?user=ford&password=omg!what&flag=true，在提出下游请求之后，它被设置为 /42?user=ford&password=***&flag=true。根据 YAML 规范，必须使用 $\ 表示 $。

## SaveSession
将调用转发到下游之前，强制执行 WebSession::save 操作。这在使用带有惰性数据存储的 Spring Session 之类的技术时特别有用，我们需要确保在进行转发调用之前保存了会话状态。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: save_session
        uri: https://example.org
        predicates:
        - Path=/foo/**
        filters:
        - SaveSession
```

如果我们将 Spring Security 与 Spring Session 集成，并希望确保安全细节已被转发到远程进程，这是非常重要的。

## SecureHeaders
SecureHeaders 在响应中添加了许多 headers。

* X-Xss-Protection:1 (mode=block)
* Strict-Transport-Security (max-age=631138519)
* X-Frame-Options (DENY)
* X-Content-Type-Options (nosniff)
* Referrer-Policy (no-referrer)
* Content-Security-Policy (default-src 'self' https:; font-src 'self' https: data:; img-src 'self' https: data:; object-src 'none'; script-src https:; style-src 'self' https: 'unsafe-inline)'
* X-Download-Options (noopen)
* X-Permitted-Cross-Domain-Policies (none)

要更改默认值，请在 spring.cloud.gateway.filter.secure-headers 命名空间中设置适当的属性。

要禁用默认值，请设置 spring.cloud.gateway.filter.secure-headers.disable 属性，属性值由逗号分隔值。示例：

```yaml
spring.cloud.gateway.filter.secure-headers.disable: x-frame-options,strict-transport-security
```

## SetPath
接受一个路径模板参数。它提供了一种通过允许模板化路径片段来操作请求路径的简单方法。它使用来自 Spring Framework 的 URI 模板。允许多个匹配段。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setpath_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment}
        filters:
        - SetPath=/{segment}
```

对于 /red/blue 的请求路径，这将在发出下游请求之前将路径设置为 /blue。

## SetRequestHeader
接受名称和值参数。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setrequestheader_route
        uri: https://example.org
        filters:
        - SetRequestHeader=X-Request-Red, Blue
```

此过滤器使用给定名称替换(而不是添加)所有 headers。因此，如果下游服务器响应一个 X-Request-Red:1234，这将被替换为 X-Request-Red:Blue，这是下游服务将接收的内容。

SetRequestHeader 知道用于匹配路径或 host 的 URI 变量。URI 变量可以在值中使用，并在运行时展开。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setrequestheader_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - SetRequestHeader=foo, bar-{segment}
```

## SetResponseHeader
接受名称和值参数。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setresponseheader_route
        uri: https://example.org
        filters:
        - SetResponseHeader=X-Response-Red, Blue
```

此过滤器使用给定名称替换(而不是添加)所有 headers。因此，如果下游服务器响应一个 X-Response-Red:1234，这将被替换为 X-Response-Red:Blue，这是网关客户端将接收到的内容。

SetResponseHeader知道用于匹配路径或主机的URI变量 知道用于匹配路径或 host 的 URI 变量。URI 变量可以在值中使用，并在运行时展开。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setresponseheader_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - SetResponseHeader=foo, bar-{segment}
```

## SetStatus
接受单个参数 status。它必须是一个有效的 Spring HttpStatus。它可以是整数值 404 或枚举的字符串表示形式：NOT_FOUND。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: setstatusstring_route
        uri: https://example.org
        filters:
        - SetStatus=BAD_REQUEST
      - id: setstatusint_route
        uri: https://example.org
        filters:
        - SetStatus=401
```

在这两种情况下，响应的 HTTP 状态都设置为 401。

我们可以配置 SetStatus GatewayFilter，以在响应的 header 中返回代理请求的原始 HTTP 状态代码。如果配置了以下属性，header 将添加到响应：

```yaml
spring:
  cloud:
    gateway:
      set-status:
        original-status-header-name: original-http-status
```

## StripPrefix
接受一个参数 parts。parts 参数表示在将请求发送到下游之前，从请求中剥离的路径中的部件数。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: nameRoot
        uri: https://nameservice
        predicates:
        - Path=/name/**
        filters:
        - StripPrefix=2
```

当通过网关向 /name/blue/red 发出请求时，向 nameservice 发出的请求看起来就像 nameservice/red。

## Retry
支持以下参数:
* retries 应该尝试的重试次数。默认 3 次。
* statuses 应该重试的 HTTP 状态代码，用 org.springframework.http.HttpStatus 表示。
* methods 应该重试的 HTTP 方法，用 org.springframework.http.HttpMethod 表示。默认 GET。
* series 重试状态码的序列，用 org.springframework.http.HttpStatus.Series 表示。默认 5XX series。
* exceptions 应该重试的抛出异常的列表。默认 IOException 和 TimeoutException。
* backoff 为重试配置的指数后退。在 firstBackoff * (factor ^ n) 的回退间隔后执行重试，其中 n 为迭代。如果配置了 maxBackoff，则应用的最大 backoff 限制为 maxBackoff。如果 basedOnPreviousValue 为 true，则使用 prevBackoff * 因子计算回退。默认 disabled。

示例：
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: retry_test
        uri: http://localhost:8080/flakey
        predicates:
        - Host=*.retry.com
        filters:
        - name: Retry
          args:
            retries: 3
            statuses: BAD_GATEWAY
            backoff:
              firstBackoff: 10ms
              maxBackoff: 50ms
              factor: 2
              basedOnPreviousValue: false
```

## RequestSize
当请求大小大于允许的限制时，此过滤器可以限制请求到达下游服务。接受RequestSize参数。它是以字节为单位定义的请求的允许大小限制。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: request_size_route
        uri: http://localhost:8080/upload
        predicates:
        - Path=/upload
        filters:
        - name: RequestSize
          args:
            maxSize: 5000000
```

此过滤器将响应状态设置为 413 负载太大，并在请求因大小而被拒绝时附加一个header errorMessage。下面的例子显示了这样一个错误消息:

```
errorMessage` : `Request size is larger than permissible limit. Request size is 6.0 MB where permissible limit is 5.0 MB
```

如果路由定义中没有提供过滤器参数，则默认请求大小设置为 5mb。

## Default Filters
要添加一个过滤器并将其应用于所有 routes，我们可以使用 spring.cloud.gateway.default-filters。此属性接受 filters 列表。示例：

```yaml
spring:
  cloud:
    gateway:
      default-filters:
      - AddResponseHeader=X-Response-Default-Red, Default-Blue
      - PrefixPath=/httpbin
```