---
title: Spring Cloud Gateway 开发指南（三） Global Filters
date: 2020-02-27 11:44:00
categories: Spring Cloud
tags:
    - Spring Cloud Gateway
---
GlobalFilter 接口具有与 GatewayFilter 接相同的签名。这些特殊的过滤器可以有条件的应用于所有的路由。（这些接口和用法在以后的版本中可能会被修改）

## Combined Global过滤器and GatewayFilter Ordering
当请求与路由匹配时，filtering web handler 会加载所有的 GlobalFilter 实例以及这个路由上配置的所有的 GatewayFilter，然后组成一个完整的过滤器链。这个合并的过滤器链使用 org.springframework.core.Ordered 接口进行排序，可以通过实现 Ordered 接口中的 getOrder() 方法（或直接使用 @Order 注解）修改过滤器的顺序。

由于 Spring Cloud Gateway 分开执行 “pre” 和 “post” 的过滤器，因此优先级最高的过滤器是 “pre” 阶段的第一个过滤器，是 “post” 阶段的最后一个过滤器。

示例：
```java
@Bean
public GlobalFilter customFilter() {
    return new CustomGlobalFilter();
}

public class CustomGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("custom global filter");
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1;
    }
}
```

## Forward Routing Filter
此过滤器在 exchange 的 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 属性中查找 URI。如果这个 URI 是 forward 模式（如 forward:///localendpoint），则使用 Spring DispatcherHandler 处理请求。请求 URL 的路径部分被转发 URL 中的路径覆盖。未修改的原始 URL 被附加到 ServerWebExchangeUtils.GATEWAY_ORIGINAL_REQUEST_URL_ATTR 属性中。

示例：
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: forward_routing_filter
        uri: forward:///app
        predicates:
        - Path=/forwardFilter
```

## LoadBalancerClient Filter
此过滤器在 exchange 的 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 属性中查找 URI。如果这个 URI 是 lb 模式（如 lb:///myservice），则使用 Spring Cloud LoadBalancerClient 将名称(本例中为 myservice)解析为实际的主机和端口，并替换相同属性中的 URI。未修改的原始 URL 被附加到 ServerWebExchangeUtils.GATEWAY_ORIGINAL_REQUEST_URL_ATTR 属性中。此过滤器也会查看 exchange 的 ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR 属性值否等于 lb。如果是，则应用相同的规则。

示例：
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: myRoute
        uri: lb://service
        predicates:
        - Path=/service/**
```

默认情况下，当在 LoadBalancer 中找不到服务实例时，将返回一个 503。可以通过设置 spring.cloud.gateway.loadbalancer.use404=true 来配置网关以返回 404。

从 LoadBalancer 返回的 ServiceInstance 的 isSecure 值，会覆盖向网关发出的请求中指定的 scheme。例如，如果请求通过 HTTPS 进入网关，但 ServiceInstance 表明它不安全，则下游请求通过 HTTP 发出。相反的情况也适用。但是，如果网关配置中为路由指定了 GATEWAY_SCHEME_PREFIX_ATTR 属性，则删除前缀，路由 URL 的结果方案将覆盖 ServiceInstance 配置。

## ReactiveLoadBalancerClientFilter
此过滤器在 exchange 的 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 属性中查找 URI。如果这个 URI 是 lb 模式（如 lb:///myservice），则使用 Spring Cloud ReactorLoadBalancer 将名称(本例中为 myservice)解析为实际的主机和端口，并替换相同属性中的 URI。未修改的原始 URL 被附加到 ServerWebExchangeUtils.GATEWAY_ORIGINAL_REQUEST_URL_ATTR 属性中。此过滤器也会查看 exchange 的 ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR 属性值否等于 lb。如果是，则应用相同的规则。

示例：
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: myRoute
        uri: lb://service
        predicates:
        - Path=/service/**
```

默认情况下，当在 ReactorLoadBalancer 中找不到服务实例时，将返回一个 503。可以通过设置 spring.cloud.gateway.loadbalancer.use404=true 来配置网关以返回 404。

从 ReactiveLoadBalancerClientFilter 返回的 ServiceInstance 的 isSecure 值，会覆盖向网关发出的请求中指定的 scheme。例如，如果请求通过 HTTPS 进入网关，但 ServiceInstance 表明它不安全，则下游请求通过 HTTP 发出。相反的情况也适用。但是，如果网关配置中为路由指定了 GATEWAY_SCHEME_PREFIX_ATTR 属性，则删除前缀，路由 URL 的结果方案将覆盖 ServiceInstance 配置。

## Netty Routing Filter
当从 exchange 的 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 属性中获取的 URL 的 scheme 是 https 或 http 时，此过滤器生效。它使用 Netty 的 HttpClient 发出下游代理请求，请求返回的结果将存储在 exchange 的 ServerWebExchangeUtils.CLIENT_RESPONSE_ATTR 属性中，过滤器链后面的过滤器可以从中获取返回的结果。（还有一个实验性的 filter:WebClientHttpRoutingFilter，它和 NettyRoutingFilter 的功能一样，但是不使用 netty。）

## Netty Write Response Filter
当 exchange 的 ServerWebExchangeUtils.CLIENT_RESPONSE_ATTR 属性中存在 HttpClientResponse 时，此过滤器生效。它在所有的其它的过滤器执行完成之后运行，将响应的数据发送给网关的客户端。 (还有一个实验性的 filter:WebClientWriteResponseFilter，它和 NettyWriteResponseFilter 的功能一样，但是不使用 netty。)

## RouteToRequestUrl Filter
当 exchange 的 ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR 属性中有一个路由对象时，此过滤器生效。它根据请求的 URI 创建一个新的 URI，但使用路由对象的 URI 属性进行更新。新的 URI 存放在 exchange 的 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR 属性中。

如果 URI 有一个 scheme 前缀，例如 lb:ws://serviceid，则从 URI 中剥离 lb 模式并将其放入 ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR 属性中，稍后在过滤器链中使用。

## The Websocket Routing Filter
当位于 excahnge 的 ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATT 属性中的 URL 有一个 ws 或 wss scheme 时，此过滤器生效。它使用 Spring WebSocket 基础设施将 WebSocket 请求转发到下游。

可以通过在 URI 前面加上 lb 来实现负载平衡，比如 lb:ws://serviceid。

示例：
```yaml
spring:
  cloud:
    gateway:
      routes:
      # SockJS route
      - id: websocket_sockjs_route
        uri: http://localhost:3001
        predicates:
        - Path=/websocket/info/**
      # Normal Websocket route
      - id: websocket_route
        uri: ws://localhost:3001
        predicates:
        - Path=/websocket/**
```

## The Gateway Metrics Filter
要启用网关指标，添加 spring-boot-starter-actuator 作为项目依赖项。
在默认情况下，只有属性 spring.cloud.gateway.metrics.enabled 不是 false，此过滤器就会生效。此过滤器会添加一个名字为 gateway.requests 和 tags 为如下的 timer metric：

* routeId 路由的 ID
* routeUri 被路由的的 URI
* outcome 被 HttpStatus.Series 标记的结果
* status 返回给客户端的 HTTP 状态码
* httpStatusCode 返回给客户端的 HTTP 状态码
* httpMethod 请求使用的 HTTP 方法

可以通过 /actuator/metrics/gateway.requests 提取这些指标，这些指标可以很容易地与 Prometheus 集成，以创建一个 Grafana dashboard。

要启用 prometheus endpoint，添加 micrometer-registry-prometheus 作为项目依赖项。

## Marking An Exchange As Routed
如果网关已经路由过一个 ServerWebExchange，它将标记这个 exchange 已经被路由过，记录在 exchange 的 ServerWebExchangeUtils.GATEWAY_ALREADY_ROUTED_ATTR 属性中。一旦被标记完成了路由，其它的路由过滤器将不会再路由本次请求，直接跳过此过滤器。有两个方便的方法，可以使用它们标记已路由过或检测是否已路由过。
* ServerWebExchangeUtils.isAlreadyRouted takes a ServerWebExchange object and checks if it has been "routed"
* ServerWebExchangeUtils.setAlreadyRouted takes a ServerWebExchange object and marks it as "routed"

## 参考资料
1. [Spring Cloud Gateway](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.1.RELEASE/reference/html/)
2. [Spring Gateway 全局过滤器 Global Filters](https://www.cnblogs.com/wgslucky/p/11632572.html)