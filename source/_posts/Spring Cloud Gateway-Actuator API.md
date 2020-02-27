---
title: Spring Cloud Gateway-Actuator API
date: 2020-02-26 18:16:00
categories: Spring Cloud
---
我们可以通过 /gateway actuator 端点来监控 Spring Cloud Gateway 应用程序并与之交互。不过要进行远程访问，必须在应用程序配置属性中启用该端点并将其暴露在 HTTP 或 JMX 上。配置如下：

```yaml
management:
  endpoint:
    gateway:
      # 默认已开启
      enabled: true
  endpoints:
    web:
      # 默认只暴露了 health/info 端点
      exposure:
        include: health,info,gateway
```

## 详细的 Actuator 格式
Spring Cloud Gateway 增加了一种新的、更详细的格式。它为每个路由添加了更多的细节，允许你查看与每个路由关联的谓词和过滤器，以及任何可用的配置。

例如，当我们访问端点 **/actuator/gateway/routes** 时，会得到类似以下响应内容：
```json
[{
	"predicate": "Paths: [/api-seckill/**], match trailing slash: true",
	"route_id": "seckill",
	"filters": ["[[StripPrefix parts = 1], order = 1]", "[org.springframework.cloud.gateway.filter.factory.RequestRateLimiterGatewayFilterFactory$$Lambda$775/831609821@27b48b61, order = 2]", "[[SpringCloudCircuitBreakerResilience4JFilterFactory name = 'circuitBreaker', fallback = forward:/fallback], order = 3]"],
	"uri": "http://localhost:8081",
	"order": 0
}, {
	"predicate": "(Paths: [/api-account/**], match trailing slash: true && gateway.routepredicatefactories.CustomRoutePredicateFactory$$Lambda$781/1730708228@7785ff79)",
	"route_id": "account",
	"filters": ["[[StripPrefix parts = 1], order = 1]", "[gateway.gatewayfilterfactories.CustomPreGatewayFilterFactory$$Lambda$783/509794966@14af153d, order = 1]", "[org.springframework.cloud.gateway.filter.factory.RequestRateLimiterGatewayFilterFactory$$Lambda$775/831609821@71ece0cb, order = 2]", "[gateway.gatewayfilterfactories.CustomPostGatewayFilterFactory$$Lambda$785/1358748845@72809214, order = 2]", "[[SpringCloudCircuitBreakerResilience4JFilterFactory name = 'circuitBreaker', fallback = forward:/fallback], order = 3]"],
	"uri": "http://localhost:8082",
	"order": 0
}]
```

默认情况下启用此功能。要禁用它，可以设置以下属性：

```yaml
spring.cloud.gateway.actuator.verbose.enabled: false
```

## 检索路由过滤器
本节详细介绍如何检索路由过滤器，包括：
* 全局过滤器
* 路由过滤器

### 全局过滤器
要检索应用于所有路由的全局过滤器，需向端点 **/actuator/gateway/globalfilters** 发送 GET 请求。得到的响应类似于以下内容：

```json
{
	"org.springframework.cloud.gateway.filter.RemoveCachedBodyFilter@217235f5": -2147483648,
	"org.springframework.cloud.gateway.filter.AdaptCachedBodyGlobalFilter@64502326": -2147482648,
	"org.springframework.cloud.gateway.filter.GatewayMetricsFilter@697a0948": 0,
	"org.springframework.cloud.gateway.filter.NettyWriteResponseFilter@79957f11": -1,
	"org.springframework.cloud.gateway.filter.WebsocketRoutingFilter@28393e82": 2147483646,
	"org.springframework.cloud.gateway.filter.ForwardPathFilter@18d47df0": 0,
	"gateway.globalfilters.ElapsedGlobalFilter@2dd1086": -2147483648,
	"org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter@4b41587d": 10000,
	"org.springframework.cloud.gateway.config.GatewayNoLoadBalancerClientAutoConfiguration$NoLoadBalancerClientFilter@7cf63b9a": 10100,
	"org.springframework.cloud.gateway.filter.ForwardRoutingFilter@4aebee4b": 2147483647,
	"org.springframework.cloud.gateway.filter.NettyRoutingFilter@6b8d54da": 2147483647
}
```

响应包含已生效的全局过滤器的详细信息。对于每个全局过滤器，有一个过滤器对象的字符串表示（例如，org.springframework.cloud.gateway.filter.LoadBalancerClientFilter@7cf63b9a）和过滤器链中相应的顺序。

### 路由过滤器
要检索应用于路由的 GatewayFilter 工厂，需向端点 /actuator/gateway/routefilters 发送 GET 请求。得到的响应类似于以下内容：

```json
{
	"[SetPathGatewayFilterFactory@26c8b296 configClass = SetPathGatewayFilterFactory.Config]": null,
	"[PreserveHostHeaderGatewayFilterFactory@1edccfd4 configClass = Object]": null,
	"[RewriteLocationResponseHeaderGatewayFilterFactory@45801322 configClass = RewriteLocationResponseHeaderGatewayFilterFactory.Config]": null,
	"[RemoveRequestHeaderGatewayFilterFactory@1efac5b9 configClass = AbstractGatewayFilterFactory.NameConfig]": null,
	"[RequestSizeGatewayFilterFactory@5b1e88f configClass = RequestSizeGatewayFilterFactory.RequestSizeConfig]": null,
	"[DedupeResponseHeaderGatewayFilterFactory@1d2fb82 configClass = DedupeResponseHeaderGatewayFilterFactory.Config]": null,
	"[AddRequestParameterGatewayFilterFactory@76c86567 configClass = AbstractNameValueGatewayFilterFactory.NameValueConfig]": null,
	"[SaveSessionGatewayFilterFactory@3520958b configClass = Object]": null,
	"[RequestRateLimiterGatewayFilterFactory@40df6090 configClass = RequestRateLimiterGatewayFilterFactory.Config]": null,
	"[RedirectToGatewayFilterFactory@8c43966 configClass = RedirectToGatewayFilterFactory.Config]": null,
	"[RemoveRequestParameterGatewayFilterFactory@11a3a45f configClass = AbstractGatewayFilterFactory.NameConfig]": null,
	"[RequestHeaderToRequestUriGatewayFilterFactory@291a4791 configClass = AbstractGatewayFilterFactory.NameConfig]": null,
	"[CustomPostGatewayFilterFactory@5a05dd30 configClass = CustomPostGatewayFilterFactory.Config]": null,
	"[StripPrefixGatewayFilterFactory@6cc64028 configClass = StripPrefixGatewayFilterFactory.Config]": null,
	"[ModifyRequestBodyGatewayFilterFactory@5a4dda2 configClass = ModifyRequestBodyGatewayFilterFactory.Config]": null,
	"[RetryGatewayFilterFactory@44d7e24 configClass = RetryGatewayFilterFactory.RetryConfig]": null,
	"[ModifyResponseBodyGatewayFilterFactory@34045582 configClass = ModifyResponseBodyGatewayFilterFactory.Config]": null,
	"[RequestHeaderSizeGatewayFilterFactory@340cb97f configClass = RequestHeaderSizeGatewayFilterFactory.Config]": null,
	"[FallbackHeadersGatewayFilterFactory@11c88cca configClass = FallbackHeadersGatewayFilterFactory.Config]": null,
	"[SpringCloudCircuitBreakerResilience4JFilterFactory@6a1568d6 configClass = SpringCloudCircuitBreakerFilterFactory.Config]": null,
	"[SecureHeadersGatewayFilterFactory@1d289d3f configClass = Object]": null,
	"[SetResponseHeaderGatewayFilterFactory@7f27f59b configClass = AbstractNameValueGatewayFilterFactory.NameValueConfig]": null,
	"[PrefixPathGatewayFilterFactory@3db65c0d configClass = PrefixPathGatewayFilterFactory.Config]": null,
	"[RewritePathGatewayFilterFactory@8c0a23f configClass = RewritePathGatewayFilterFactory.Config]": null,
	"[RemoveResponseHeaderGatewayFilterFactory@69796bd0 configClass = AbstractGatewayFilterFactory.NameConfig]": null,
	"[MapRequestHeaderGatewayFilterFactory@250d440 configClass = MapRequestHeaderGatewayFilterFactory.Config]": null,
	"[AddRequestHeaderGatewayFilterFactory@dbed7fd configClass = AbstractNameValueGatewayFilterFactory.NameValueConfig]": null,
	"[AddResponseHeaderGatewayFilterFactory@7e5efcab configClass = AbstractNameValueGatewayFilterFactory.NameValueConfig]": null,
	"[CustomPreGatewayFilterFactory@1b52699c configClass = CustomPreGatewayFilterFactory.Config]": null,
	"[SetRequestHeaderGatewayFilterFactory@10f405ff configClass = AbstractNameValueGatewayFilterFactory.NameValueConfig]": null,
	"[RewriteResponseHeaderGatewayFilterFactory@1c98b4eb configClass = RewriteResponseHeaderGatewayFilterFactory.Config]": null,
	"[SetStatusGatewayFilterFactory@756b2d90 configClass = SetStatusGatewayFilterFactory.Config]": null
}
```

响应包含应用于任何特定路由的 GatewayFilter 工厂的详细信息。每个工厂都有对应对象的字符串表示(例如，[SecureHeadersGatewayFilterFactory@1d289d3f configClass = object])。注意，null 值是由于端点控制器的实现不完整造成的，因为它试图在过滤器链中设置对象的顺序，而这并不适用于 GatewayFilter 工厂对象。

## 刷新路由缓存
若要清除路由缓存，需向端点 **/actuator/gateway/refresh** 发送 POST 请求。请求返回一个没有响应体的 200。

## 检索网关中定义的路由
要检索网关中定义的路由，需向端点 /actuator/gateway/routes 发送 GET 请求。得到的响应类似于以下内容：

```json
[{
	"predicate": "Paths: [/api-seckill/**], match trailing slash: true",
	"route_id": "seckill",
	"filters": ["[[StripPrefix parts = 1], order = 1]", "[org.springframework.cloud.gateway.filter.factory.RequestRateLimiterGatewayFilterFactory$$Lambda$775/831609821@27b48b61, order = 2]", "[[SpringCloudCircuitBreakerResilience4JFilterFactory name = 'circuitBreaker', fallback = forward:/fallback], order = 3]"],
	"uri": "http://localhost:8081",
	"order": 0
}, {
	"predicate": "(Paths: [/api-account/**], match trailing slash: true && gateway.routepredicatefactories.CustomRoutePredicateFactory$$Lambda$781/1730708228@7785ff79)",
	"route_id": "account",
	"filters": ["[[StripPrefix parts = 1], order = 1]", "[gateway.gatewayfilterfactories.CustomPreGatewayFilterFactory$$Lambda$783/509794966@14af153d, order = 1]", "[org.springframework.cloud.gateway.filter.factory.RequestRateLimiterGatewayFilterFactory$$Lambda$775/831609821@71ece0cb, order = 2]", "[gateway.gatewayfilterfactories.CustomPostGatewayFilterFactory$$Lambda$785/1358748845@72809214, order = 2]", "[[SpringCloudCircuitBreakerResilience4JFilterFactory name = 'circuitBreaker', fallback = forward:/fallback], order = 3]"],
	"uri": "http://localhost:8082",
	"order": 0
}]
```

响应包含网关中定义的所有路由的详细信息。下表描述了响应的每个元素(每个都是一个路由)的结构：

属性名 | 属性类型 | 属性描述
:-: | :-: | :-:
route_id | String | The route ID.
predicate | Object | The route predicate.
filters | Array | The GatewayFilter factories applied to the route.
uri | String | The destination URI of the route..
order | Number | The route order.

## 检索指定定路由的信息
要检索关于单个路由的信息，需向端点 /actuator/gateway/routes/{id} 发送 GET 请求（例如，/actuator/gateway/routes/account）。得到的响应类似于以下内容：

```json
{
	"predicate": "(Paths: [/api-account/**], match trailing slash: true && gateway.routepredicatefactories.CustomRoutePredicateFactory$$Lambda$781/1730708228@7785ff79)",
	"route_id": "account",
	"filters": ["[[StripPrefix parts = 1], order = 1]", "[gateway.gatewayfilterfactories.CustomPreGatewayFilterFactory$$Lambda$783/509794966@14af153d, order = 1]", "[org.springframework.cloud.gateway.filter.factory.RequestRateLimiterGatewayFilterFactory$$Lambda$775/831609821@71ece0cb, order = 2]", "[gateway.gatewayfilterfactories.CustomPostGatewayFilterFactory$$Lambda$785/1358748845@72809214, order = 2]", "[[SpringCloudCircuitBreakerResilience4JFilterFactory name = 'circuitBreaker', fallback = forward:/fallback], order = 3]"],
	"uri": "http://localhost:8082",
	"order": 0
}
```

下表描述了响应的结构：

属性名 | 属性类型 | 属性描述
:-: | :-: | :-:
route_id | String | The route ID.
predicate | Object | The route predicate.
filters | Array | The GatewayFilter factories applied to the route.
uri | String | The destination URI of the route..
order | Number | The route order.

## 创建和删除指定路由
要创建路由，需向端点 /gateway/routes/{id_route_to_create} 发送 POST 请求并携带 JSON body。JSON body 类似于以下内容：

```json
{
  "id": "first_route",
  "predicates": [{
    "name": "Path",
    "args": {"_genkey_0":"/first"}
  }],
  "filters": [],
  "uri": "https://www.uri-destination.org",
  "order": 0
}]
```

要删除路由，需向端点 /gateway/routes/{id_route_to_delete} 发送 DELETE 请求。

## 重述：所有端点列表
下面的表格总结了 Spring Cloud Gateway actuator 的所有端点(注意每个端点都有 /actuator/gateway 作为基本路径)：

ID | HTTP Method | Description
:-: | :-: | :-:
globalfilters | GET | Displays the list of global filters applied to the routes.
routefilters | GET | Displays the list of GatewayFilter factories applied to a particular route.
refresh | POST | Clears the routes cache.
routes | GET | Displays the list of routes defined in the gateway.
routes/{id} | GET | Displays information about a particular route.
routes/{id} | POST | Adds a new route to the gateway.
routes/{id} | DELETE | Removes an existing route from the gateway.

