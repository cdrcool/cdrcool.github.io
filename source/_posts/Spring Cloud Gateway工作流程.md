---
title: Spring Cloud Gateway 工作流程
date: 2020-02-26 11:22:00
categories: Spring Cloud
---
## 术语
* Route(路由)：网关的基本构件。它由 ID、目标 URI、Predicates 和 Filters 定义。如果聚合 Predicate 为真，则匹配路由。
* Predicate(谓词)：Java 8 Function Predicate。输入类型是一个 Spring Framework ServerWebExchange。它允许开发人员匹配 HTTP 请求中的任何内容，比如头或参数。
* Filter(过滤器)：用特定工厂构造的 Spring Framework GatewayFilter 实例。在这里，可以在发送下游请求之前或之后修改请求和响应。

## 工作流程
![Spring Cloud Gateway 工作流程](/images/springcloud/Spring%20Cloud%20Gateway工作流程.png)

1. 客户端向 Spring Cloud Gateway 发出请求。
2. 如果 Gateway Handler Mapping 确定请求与某个 Route 匹配，则将其发送给 Gateway Web Handler。
3. 该 Handler 通过特定于请求的 Filter Chain 运行请求。Filters 被虚线分隔的原因是 Filters 可以在发送代理请求之前和之后运行逻辑。执行所有 Pre Filters 逻辑。然后发出代理请求。发出代理请求后，将运行 Post Filters 逻辑。