---
title: Spring Security 参考文档-全局
date: 2020-03-06 11:25:00
categories: Spring Security
tags:
    - Spring Security
---
本文讨论 Spring Security 在基于 Servlet 的应用程序中的高级体系结构。

## 过滤器综述
Spring Security 对 Servlet 的支持是基于 Servlet过滤器的，因此通常首先查看过滤器的角色是有帮助的。下图显示了单个 HTTP 请求的处理程序的典型分层。

![filterchain](/images/springsecurity/filterchain.png)

客户端向应用程序发送请求，然后容器创建一个 FilterChain，其中包含应该基于请求 URI 的路径处理 HttpServletRequest 的过滤器和 Servlet。在 Spring MVC 应用程序中，Servlet 是 DispatcherServlet 的一个实例。最多一个 Servlet 可以处理一个 HttpServletRequest 和 HttpServletResponse。但是，可以使用多个过滤器：

* 防止下游过滤器或 Servlet 被调用。在这种情况下，过滤器通常会编写 HttpServletResponse。
* 修改下游过滤器和 Servlet 使用的 HttpServletRequest 或 HttpServletResponse。


过滤器的能力来自于传递给它的 FilterChain。

**FilterChain 使用示例：**
```java
public void doFilter(ServletRequest request， ServletResponse response， FilterChain chain) {
    // do something before the rest of the application
    chain.doFilter(request， response); // invoke the rest of the application
    // do something after the rest of the application
}
```

**由于过滤器只影响下游的过滤器和 Servlet，因此调用每个过滤器的顺序非常重要。**

## DelegatingFilterProxy
Spring 提供了一个名为 DelegatingFilterProxy 的过滤器实现，它允许在 Servlet 容器的生命周期和 Spring 的 ApplicationContext 之间建立桥接。Servlet 容器允许使用自己的标准注册过滤器，但它不知道 Spring 定义的 Beans。**DelegatingFilterProxy 可以通过标准的 Servlet 容器机制注册，但是将所有工作委托给实现过滤器的 Spring Bean。**

下图演示了 DelegatingFilterProxy 如何融入过滤器和 FilterChain。

![delegatingfilterproxy](/images/springsecurity/delegatingfilterproxy.png)

DelegatingFilterProxy 从 ApplicationContext 中查找 Bean Filter0，然后调用 Bean Filter0。下面是 DelegatingFilterProxy 的伪代码。

```java
public void doFilter(ServletRequest request， ServletResponse response， FilterChain chain) {
    // Lazily get Filter that was registered as a Spring Bean
    // For the example in DelegatingFilterProxy delegate is an instance of Bean Filter0
    Filter delegate = getFilterBean(someBeanName);
    // delegate work to the Spring Bean
    delegate.doFilter(request， response);
}
```

DelegatingFilterProxy 的另一个好处是，它允许延迟查找过滤器 bean 实例。这很重要，因为容器需要在启动之前注册过滤器实例。然而，Spring 通常使用 ContextLoaderListener 来加载 Spring Beans，直到需要注册过滤器实例之后才会这样做。

##  FilterChainProxy
Spring Security 的 Servlet 支持包含在 FilterChainProxy 中。FilterChainProxy 是 Spring Security 提供的一个特殊过滤器，它允许通过 SecurityFilterChain 将多个过滤器实例委托给它。由于 FilterChainProxy 是一个 Bean，它通常被包装在一个 DelegatingFilterProxy 中。

![filterchainproxy](/images/springsecurity/filterchainproxy.png)

## SecurityFilterChain
FilterChainProxy 使用 SecurityFilterChain 来确定应该为这个请求调用哪些 Spring Security 过滤器。

![securityfilterchain](/images/springsecurity/securityfilterchain.png)

SecurityFilterChain 中的 Security 过滤器通常是 Beans，但是它们是在 FilterChainProxy 中注册的，而不是 DelegatingFilterProxy。FilterChainProxy 为直接向 Servlet 容器或 DelegatingFilterProxy 注册提供了许多优势。首先，**它为所有 Spring Security 的 Servlet 支持提供了一个起点**。出于这个原因，如果我们试图排除 Spring Security 的 Servlet 支持的故障，那么在 FilterChainProxy 中添加一个调试点是一个很好的开始。

其次，由于 FilterChainProxy 是 Spring Security 使用的核心，**它可以执行非可选的任务**。例如，它清除 SecurityContext 以避免内存泄漏。它还应用 Spring Security 的 HttpFirewall 来保护应用程序免受某些类型的攻击。

此外，它在确定何时应调用 SecurityFilterChain 时提供了更大的灵活性。在 Servlet 容器中，仅根据 URL 调用过滤器。但是，**FilterChainProxy 可以利用 RequestMatcher 接口根据 HttpServletRequest 中的任何内容确定调用**。

事实上，**FilterChainProxy 可以用来决定应该使用哪个 SecurityFilterChain。这允许在应用程序中为不同的片提供完全独立的配置**。

![multi-securityfilterchain](/images/springsecurity/multi-securityfilterchain.png)

在 Multiple SecurityFilterChain 中，FilterChainProxy 决定应该使用哪个 SecurityFilterChain。只有第一个匹配的 SecurityFilterChain 才会被调用。如果请求一个 /api/messages/ 的 URL，它将首先匹配 SecurityFilterChain0 的 /api/** 模式，所以只有 SecurityFilterChain0 将被调用，即使它也匹配 SecurityFilterChainn。如果请求一个 /messages/ 的 URL，它将与 SecurityFilterChain0的/api/** 模式不匹配，因此 FilterChainProxy 将继续尝试每个 SecurityFilterChain。假设没有其他的，与 SecurityFilterChainn 相匹配的 SecurityFilterChain 实例将被调用。

注意 SecurityFilterChain0 只配置了三个安全过滤器实例。但是，SecurityFilterChainn 配置了四个安全过滤器。需要注意的是，每个 SecurityFilterChain 都可以是唯一的，并且可以单独配置。事实上，如果应用程序希望 Spring Security 忽略某些请求，SecurityFilterChain 可能没有安全过滤器。

## Security Filters
使用 SecurityFilterChain API 将 Security 过滤器插入到 FilterChainProxy中。过滤器的顺序很重要。通常不需要知道 Spring Security 过滤器的顺序。然而，有时了解顺序是有益的：

以下是 Spring Security Filter ordering 的一个全面的清单：
* ChannelProcessingFilter
* ConcurrentSessionFilter
* WebAsyncManagerIntegrationFilter
* SecurityContextPersistenceFilter
* HeaderWriterFilter
* CorsFilter
* CsrfFilter
* LogoutFilter
* OAuth2AuthorizationRequestRedirectFilter
* Saml2WebSsoAuthenticationRequestFilter
* X509AuthenticationFilter
* AbstractPreAuthenticatedProcessingFilter
* CasAuthenticationFilter
* OAuth2LoginAuthenticationFilter
* Saml2WebSsoAuthenticationFilter
* UsernamePasswordAuthenticationFilter
* ConcurrentSessionFilter
* OpenIDAuthenticationFilter
* DefaultLoginPageGeneratingFilter
* DefaultLogoutPageGeneratingFilter
* DigestAuthenticationFilter
* BearerTokenAuthenticationFilter
* BasicAuthenticationFilter
* RequestCacheAwareFilter
* SecurityContextHolderAwareRequestFilter
* JaasApiIntegrationFilter
* RememberMeAuthenticationFilter
* AnonymousAuthenticationFilter
* OAuth2AuthorizationCodeGrantFilter
* SessionManagementFilter
* ExceptionTranslationFilter
* FilterSecurityInterceptor
* SwitchUserFilter

## 处理安全异常
ExceptionTranslationFilter 允许将 AccessDeniedException 和 AuthenticationException 转换为 HTTP 响应。

ExceptionTranslationFilter 作为 Security 过滤器之一插入到 FilterChainProxy 中。

![exceptiontranslationfilter](/images/springsecurity/exceptiontranslationfilter.png)

1. 首先，ExceptionTranslationFilter 调用 FilterChain.doFilter(request， response) 来调用应用程序的其余部分。
2. 如果用户没有通过身份验证，或者是 AuthenticationException，那么启动身份验证。
    + SecurityContextHolder 删除。
    + HttpServletRequest 保存在 RequestCache 中。当用户成功进行身份验证时，将使用 RequestCache 重放原始请求。
    + AuthenticationEntryPoint 用于从客户端请求凭据。例如，它可能会重定向到一个登录页面或发送一个 WWW-Authenticate header。
3. 否则，如果是 AccessDeniedException，则 Access Denied。调用 AccessDeniedHandler 来处理拒绝的访问。

如果应用程序没有抛出 AccessDeniedException 或 AuthenticationException，则 ExceptionTranslationFilter不执行任何操作。

ExceptionTranslationFilter 的伪代码如下所示：

```java
try {
    filterChain.doFilter(request， response); 
} catch (AccessDeniedException | AuthenticationException e) {
    if (!authenticated || e instanceof AuthenticationException) {
        startAuthentication(); 
    } else {
        accessDenied(); 
    }
}
```

* 回顾一下过滤器，会发现调用 FilterChain.doFilter(request, response) 相当于调用应用程序的其余部分。这意味着，如果应用程序的另一部分(即 FilterSecurityInterceptor 或 method security)抛出 AuthenticationException 或 AccessDeniedException，它将在这里捕获并处理。
* 如果用户没有通过身份验证，或者是 AuthenticationException，则启动身份验证。
* 否则，拒绝访问。

## 补充
### Spring Security 过滤器链
#### UML 类图
![Spring Security 过滤器链 UML](/images/springsecurity/Spring%20Security过滤器链UML.png)

#### 默认过滤器链
**表单认证：**
![Spring Security过滤器链-表单认证](/images/springsecurity/Spring%20Security过滤器链-表单认证.png)

**Http Basic 认证：**
![Spring Security过滤器链-Http Basic 认证](/images/springsecurity/Spring%20Security过滤器链-Http%20Basic认证.png)

## 参考资料
1. [Servlet Security: The Big Picture](https://docs.spring.io/spring-security/site/docs/5.3.0.RELEASE/reference/html5/#servlet-architecture)