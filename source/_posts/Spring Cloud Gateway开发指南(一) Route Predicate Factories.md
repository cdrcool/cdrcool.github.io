
---
title: Spring Cloud Gateway 开发指南（一） Route Predicate Factories
date: 2020-02-27 11:22:00
categories: Spring Cloud
tags:
    - Spring Spring Cloud Gateway
---
Spring Cloud Gateway 包含许多内置的路由 predicate factories。所有这些 predicates 都匹配 HTTP 请求的不同属性。我们可以将多个路由 predicate factories 与逻辑和语句组合在一起。

## After
接受一个日期类型参数。此 Predicate 匹配在指定日期时间之后发生的请求。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
```

该路由与丹佛时间 2017年1月20日17点42分 之后的任何请求相匹配。

## Before
接受一个日期类型参数。此 Predicate 匹配在指定日期时间之前发生的请求。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: before_route
        uri: https://example.org
        predicates:
        - Before=2017-01-20T17:42:47.789-07:00[America/Denver]
```

该路由与丹佛时间 2017年1月20日17点42分 之前的任何请求相匹配。

## Between
接受两个个日期类型参数：datetime1 和 datetime2。此 Predicate 匹配在 datetime1 之后和 datetime2 之前发生的请求。datetime2 参数必须位于 datetime1 之后。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: between_route
        uri: https://example.org
        predicates:
        - Between=2017-01-20T17:42:47.789-07:00[America/Denver]，2017-01-21T17:42:47.789-07:00[America/Denver]
```

该路由与丹佛时间 2017年1月20日17点42分 之后和丹佛时间 2017年1月21日17:42 之前的任何请求匹配。这对于维护窗口可能很有用。

## Cookie
接受两个参数：cookie 名称和一个正则表达式。此 Predicate 匹配具有给定名称且其值与正则表达式匹配的 cookie。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: cookie_route
        uri: https://example.org
        predicates:
        - Cookie=chocolate，ch.p
```

该路由匹配具有一个名为 chocolate 的 cookie 的请求，该 cookie 的值与 ch.p 正则表达式匹配。

## Header
接受两个参数：header 名称和一个正则表达式。此 Predicate 与具有给定名称(其值与正则表达式匹配)的 header 匹配。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: https://example.org
        predicates:
        - Header=X-Request-Id，\d+
```

如果请求的 header 名为 X-Request-Id，其值与 \d+ 正则表达式匹配(即，它的值为一个或多个数字)，则该路由将进行匹配。

## Host
接受一个参数：host name patterns 列表。此 Predicate 与与模式匹配的 Host header 匹配。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Host=**.somehost.org,**.anotherhost.org
```

还支持 URI 模板变量(例如 {sub}.myhost.org)。

如果请求的 Host header 的值为 www .somehost.org 或 beta.somehost.org 或 www .anotherhost.org，则该路由将进行匹配。

此 Predicate 将 URI 模板变量(如前面示例中定义的 sub)提取为名称和值的映射，并将其放在 ServerWebExchange.getAttributes() 中，并在 ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE 中定义一个键。然后这些值可供 GatewayFilter 工厂使用。

## Method
接受一个或多个参数：要匹配的 HTTP 方法。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: https://example.org
        predicates:
        - Method=GET,POST
```

如果请求方法是 GET 或 POST，则该路由匹配。

## Path
接受两个参数:一个 Spring PathMatcher 模式列表和一个名为 matchOptionalTrailingSeparator 的可选标志。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment},/blue/{segment}
```

如果请求路径为，例如: /red/1 或 /red/blue 或 /blue/green，则该路由匹配。

此 Predicate 将 URI 模板变量(如前面示例中定义的片段)提取为名称和值的映射，并将其放入 ServerWebExchange.getAttributes() 中，并在 ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE 中定义了一个键。然后这些值可供 GatewayFilter 工厂使用。

可以使用实用程序方法(称为 get)来简化对这些变量的访问。下面的例子展示了如何使用 get 方法：

```java
Map<String，String> uriVariables = ServerWebExchangeUtils.getPathPredicateVariables(exchange);

String segment = uriVariables.get("segment");
```

## Query
接受两个参数：一个必需的参数和一个可选的 regexp。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: https://example.org
        predicates:
        - Query=green
```

如果请求包含 green 查询参数，则前面的路由匹配。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: https://example.org
        predicates:
        - Query=red，gree.
```

如果请求包含一个 red 查询参数，其值与 gree. 正则表达式 匹配，则前面的路由匹配。

## RemoteAddr
接受 CIDR-notation(IPv4 或 IPv6) 字符串列表(最小大小为 1)，比如 192.168.0.1/16 (其中 192.168.0.1 是一个 IP 地址，16 是子网掩码)。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: https://example.org
        predicates:
        - RemoteAddr=192.168.1.1/24
```

如果请求的远程地址是 192.168.1.10，则该路由匹配。

## Weight
有两个参数：组和权值。每组计算权重。示例：

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: weight_high
        uri: https://weighthigh.org
        predicates:
        - Weight=group1，8
      - id: weight_low
        uri: https://weightlow.org
        predicates:
        - Weight=group1，2
```

该路由将 80% 的流量转发到 weighthigh.org，20%  的流量转发到 weightlow.org。