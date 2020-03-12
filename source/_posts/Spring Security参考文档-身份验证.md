---
title: Spring Security 参考文档-身份验证
date: 2020-03-06 19:34:00
categories: Spring Security
tags:
    - Spring Security
---
Spring Security 为身份验证提供了全面的支持。本文讨论：

**架构组件**

本节描述了 Spring Security 用于 Servlet 身份验证的主要体系结构组件。

* SecurityContextHolder - SecurityContextHolder 是 Spring Security 存储身份验证者详细信息的地方。
* SecurityContext - 从 SecurityContextHolder 中获取，包含当前已验证用户的 Authentication。 
* Authentication - 可以是 AuthenticationManager 的输入，以提供用户提供的用于身份验证的凭据，也可以是 SecurityContext 中的当前用户。
* GrantedAuthority - 在身份验证中授予主体的权限（即角色、作用域等）。
* AuthenticationManager - 定义 Spring Security 过滤器如何执行身份验证的 API。
* ProviderManager - AuthenticationManager 最常见的实现。
* AuthenticationProvider - 由 ProviderManager 用于执行特定类型的身份验证。
* 使用 AuthenticationEntryPoint 请求凭据 - 用于从客户端请求凭据（如重定向到登录页面，发送 WWW-Authenticate 响应，等等）。
* AbstractAuthenticationProcessingFilter - 用于身份验证的基本过滤器。这也很好地说明了身份验证的高级流程以及各个部分是如何协同工作的。

**身份验证机制**
* Username and Password - 如何使用用户名/密码进行身份验证。
* OAuth 2.0 Login - OAuth 2.0 登录使用 OpenID Connect 和非标准 OAuth 2.0 登录（即GitHub）。
* SAML 2.0 Login - SAML 2.0 登录。
* Central Authentication Server (CAS) - 中央认证服务器(CAS)支持。
* Remember Me - 如何记住用户超出会话过期时间。
* JAAS Authentication - JAAS 身份验证。
* OpenID - OpenID 身份验证(不要与 OpenID Connect 混淆)
* Pre-Authentication Scenarios - 使用外部机制（如 SiteMinder 或 Java EE security）进行身份验证，但仍使用 Spring Security 进行授权和防止常见攻击。
* X509 Authentication - X509 身份验证。

## SecurityContextHolder
Spring Security 的身份验证模型的核心是 SecurityContextHolder。它包含 SecurityContext。

![securitycontextholder](/images/springsecurity/securitycontextholder.png)

SecurityContextHolder 是 Spring Security 存储身份验证者详细信息的地方。Spring Security 并不关心 SecurityContext 如何填充。如果它包含一个值，那么它将被用作当前经过身份验证的用户。

指示用户已通过身份验证的最简单方法是直接设置 SecurityContextHolder。

```java
SecurityContext context = SecurityContextHolder.createEmptyContext(); 
Authentication authentication =
    new TestingAuthenticationToken("username", "password", "ROLE_USER"); 
context.setAuthentication(authentication);

SecurityContextHolder.setContext(context); 
```

1. 我们首先创建一个空的 SecurityContext。创建一个新的 SecurityContext 实例而不是使用 SecurityContextHolder.getContext().setAuthentication(authentication) 来避免多线程的竞争条件是很重要的。

2. 接下来，我们创建一个新的 Authentication 对象。Spring Security 并不关心在 SecurityContext 上设置了什么类型的身份验证实现。这里我们使用 TestingAuthenticationToken，因为它非常简单。更常见的生产场景是 UsernamePasswordAuthenticationToken(userDetails, password, authorities)。

3. 最后，我们在 SecurityContextHolder 上设置 SecurityContext。Spring Security 将使用这些信息进行授权。

如果希望获得关于经过身份验证的主体的信息，可以通过访问 SecurityContextHolder 来实现。

```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
String username = authentication.getName();
Object principal = authentication.getPrincipal();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
```

**默认情况下，SecurityContext 使用一个 ThreadLocal 来存储这些细节**，这意味着 SecurityContext 始终对同一执行线程中的方法可用，即使 SecurityContext 没有作为参数显式地传递给这些方法。如果在当前主体的请求被处理后小心地清除线程，那么以这种方式使用 ThreadLocal 是非常安全的。Spring Security 的 FilterChainProxy 确保总是清除 SecurityContext。

有些应用程序并不完全适合使用 ThreadLocal，因为它们使用线程的特定方式。例如，Swing 客户端可能希望 Java 虚拟机中的所有线程使用相同的安全上下文。SecurityContext 可以在启动时配置一个策略来指定希望如何存储上下文。对于一个独立的应用程序，将使用 SecurityContextHolder.MODE_GLOBAL 策略。其他应用程序可能希望安全线程派生的线程也具有相同的安全标识。这是通过使用 SecurityContextHolder.MODE_INHERITABLETHREADLOCAL 来实现的。我们有两种方式可以从默认的 SecurityContextHolder.MODE_THREADLOCAL 中更改模式。第一种是设置一个系统属性，第二种是调用 SecurityContextHolder 中的一个静态方法。大多数应用程序不需要更改默认值。

## SecurityContext
SecurityContext 是从 SecurityContextHolder 中获取的。SecurityContext 包含 Authentication 对象。

## Authentication
在 Spring Security 中，Authentication 有两个主要目的：
* AuthenticationManager 的输入，用于提供用户提供的用于身份验证的凭据。在此场景中使用时，isAuthenticated() 返回 false。
* 表示当前经过身份验证的用户。可以从 SecurityContext 中获取当前身份验证。

Authentication 包含:
* principal(主体) - 标识用户。在使用用户名/密码进行身份验证时，这通常是 UserDetails 的一个实例。
* credentials(凭证) - 通常是密码。在许多情况下，在对用户进行身份验证以确保其不被泄漏之后，会清除此信息。
* authorities(权限) - GrantedAuthoritys 是授予用户的高级权限。一些例子是 roles(角色) 或 scopes(范围)。

## GrantedAuthority
GrantedAuthoritys 是授予用户的高级权限。一些例子是角色或范围。

可以通过 Authentication.getAuthorities() 方法获得 GrantedAuthoritys。此方法提供 GrantedAuthority 对象的集合。GrantedAuthority 是授予主体的权力，这并不奇怪。这些权限通常是“角色”，例如 ROLE_ADMINISTRATOR 或 ROLE_HR_SUPERVISOR。稍后将为 web 授权、方法授权和域对象授权配置这些角色。Spring Security 的其他部分能够解读这些权限，并期待它们的出现。当使用基于用户名/密码的身份验证时，GrantedAuthoritys 通常由 UserDetailsService 加载。

通常，GrantedAuthority 对象是应用程序范围的权限。它们并不特定于给定的域对象。因此，我们可能没有 GrantedAuthority 来表示对 Employee 对象编号 54 的权限，因为如果有成千上万个这样的权限，那么很快就会耗尽内存（或者，至少会导致应用程序需要很长时间来验证用户）。当然，Spring Security 是专门为处理这一常见需求而设计的，但是我们应该使用项目的域对象安全功能来实现这一目的。

## AuthenticationManager
AuthenticationManager 是定义 Spring Security 过滤器如何执行身份验证的 API。然后，由调用 AuthenticationManager 的控制器（如 Spring Security的过滤器）在 SecurityContextHolder 上设置返回的 Authentication 。如果没有与 Spring Security 的过滤器集成，可以直接设置 SecurityContextHolder，不需要使用 AuthenticationManager。

虽然 AuthenticationManager 的实现可以是任何东西，但最常见的实现是 ProviderManager。

## ProviderManager
ProviderManager 是 AuthenticationManager 最常用的实现。ProviderManager 委托给 AuthenticationProviders 列表。每个 AuthenticationProvider 都有机会指示身份验证应成功、失败，或指示它无法做出决定，并允许下游的 AuthenticationProvider 做出决定。如果配置的 AuthenticationProvider 都不能进行身份验证，则身份验证将失败，并抛出 ProviderNotFoundException，这是一个特殊的 AuthenticationException，表示 ProviderManager 不支持传递给它的 Authentication 类型。

![providermanager](/images/springsecurity/providermanager.png)

实际上，每个 AuthenticationProvider 都知道如何执行特定类型的身份验证。例如，一个 AuthenticationProvider 可能能够验证用户名/密码，而另一个可能能够验证 SAML 断言。这允许每个 AuthenticationProvider 执行非常特定的身份验证类型，同时支持多种类型的身份验证，并且只公开一个 AuthenticationManager bean。

ProviderManager 还允许配置可选的父 AuthenticationManager，如果没有 AuthenticationProvider 可以执行身份验证，就会咨询它。父类可以是任何类型的 AuthenticationManager，但它通常是 ProviderManager 的实例。

![providermanager-parent](/images/springsecurity/providermanager-parent.png)

实际上，多个 ProviderManager 实例可能共享同一个父 AuthenticationManager。这在有多个 SecurityFilterChain 实例的场景中比较常见，这些实例有一些共同的身份验证（共享的父 AuthenticationManager），但也有不同的身份验证机制（不同的 ProviderManager实例）。

![providermanagers-parent](/images/springsecurity/providermanagers-parent.png)

默认情况下，ProviderManager 将尝试从成功的身份验证请求返回的 Authentication 对象中清除任何敏感的凭据信息。这可以防止密码等信息在 HttpSession 中保留的时间超过必要的时间。

当使用用户对象的缓存来提高无状态应用程序的性能时，这可能会导致问题。如果 Authentication 包含对缓存中某个对象的引用(例如 UserDetails 实例)，并且该对象的凭据已被删除，那么将无法根据缓存的值进行身份验证。如果使用缓存，则需要考虑到这一点。一个明显的解决方案是，首先在缓存实现中或在创建返回的 Authentication 对象的 AuthenticationProvider 中创建对象的副本。或者，可以禁用 ProviderManager 的 eraseCredentialsAfterAuthentication 属性。

## AuthenticationProvider
可以将多个 AuthenticationProvider 注入到 ProviderManager 中。每个 AuthenticationProvider 执行特定类型的身份验证。例如，DaoAuthenticationProvider 支持基于用户名/密码的身份验证，而 JwtAuthenticationProvider 支持对 JWT 令牌进行身份验证。

## 使用 AuthenticationEntryPoint 请求凭据
AuthenticationEntryPoint 用于发送从客户端请求凭据的 HTTP 响应。

有时，客户端会主动包含凭据，如用于请求资源的用户名/密码。在这些情况下，Spring Security 不需要提供一个HTTP响应来请求来自客户机的凭据，因为它们已经包含在内了。

在其他情况下，客户端将向未经授权访问的资源发出未经身份验证的请求。在这种情况下，AuthenticationEntryPoint 的实现用于从客户端请求凭据。AuthenticationEntryPoint 实现可以重定向到登录页面，使用 WWW-Authenticate header 响应，等等。

## AbstractAuthenticationProcessingFilter
AbstractAuthenticationProcessingFilter 用作验证用户凭证的基本过滤器。在认证凭证之前，Spring Security 通常使用 AuthenticationEntryPoint 请求凭证。

接下来，AbstractAuthenticationProcessingFilter 可以对提交给它的任何身份验证请求进行身份验证。

![abstractauthenticationprocessingfilter](/images/springsecurity/abstractauthenticationprocessingfilter.png)

1. 当用户提交其凭据时，AbstractAuthenticationProcessingFilter 将从要进行身份验证的 HttpServletRequest 创建 Authentication。创建的 Authentication 的类型取决于 AbstractAuthenticationProcessingFilter 的子类。例如，UsernamePasswordAuthenticationFilter 根据在 HttpServletRequest 中提交的用户名和密码创建 UsernamePasswordAuthenticationToken。

2. 接下来，将 Authentication 传递到 AuthenticationManager 以进行身份验证。

3. 如果验证失败，则失败
    + SecurityContextHolder 清除。
    + RememberMeServices.loginFail 被调用。如果没有配置 remember me，这是一个 no-op。
    + AuthenticationFailureHandler 被调用。

4. 如果认证成功，则成功。
    + SessionAuthenticationStrategy 收到新登录的通知。
    + 在 SecurityContextHolder 中设置 Authentication。稍后 SecurityContextPersistenceFilter 将 SecurityContext 保存到 HttpSession 中。
    + RememberMeServices.loginSuccess 被调用。如果没有配置 remember me，这是一个 no-op。
    + ApplicationEventPublisher 发布 InteractiveAuthenticationSuccessEvent。
    
## 用户名/密码身份验证
验证用户身份最常用的方法之一是验证用户名和密码。因此，Spring Security 为使用用户名和密码进行身份验证提供了全面的支持。

**读取用户名和密码**

Spring Security 提供了以下内置机制，用于从 HttpServletRequest 读取用户名和密码：
* 表单登录
* Basic 身份验证
* Digest 身份验证

**存储机制**
每种支持的读取用户名和密码的机制都可以利用任何支持的存储机制：
* 具有内存内身份验证的简单存储
* 具有 JDBC 身份验证的关系数据库
* 带有 UserDetailsService 的自定义数据存储
* 具有 LDAP 身份验证的 LDAP 存储

### 表单登录
Spring Security 支持通过 html 表单提供用户名和密码。本节详细介绍了Spring Security中基于表单的身份验证的工作原理。

让我们看看在 Spring Security 中基于表单的登录是如何工作的。首先，我们将看到如何将用户重定向到登录表单。

![loginurlauthenticationentrypoint](/images/springsecurity/loginurlauthenticationentrypoint.png)

1. 首先，用户向未经授权的 resource /private 发出未经身份验证的请求。

2. Spring Security 的 FilterSecurityInterceptor 表示通过抛出 AccessDeniedException 来拒绝未经身份验证的请求。

3. 因为用户没有经过身份验证，ExceptionTranslationFilter 将启动身份验证，并使用配置的 AuthenticationEntryPoint 发送重定向到登录页。在大多数情况下，AuthenticationEntryPoint 是 LoginUrlAuthenticationEntryPoint 的实例。

4. 然后浏览器将请求重定向到的登录页。

5. 在应用程序中，必须呈现登录页面。

提交用户名和密码时，UsernamePasswordAuthenticationFilter 将验证用户名和密码。UsernamePasswordAuthenticationFilter 扩展了 AbstractAuthenticationProcessingFilter，所以这个关系图应该看起来非常类似。

![usernamepasswordauthenticationfilter](/images/springsecurity/usernamepasswordauthenticationfilter.png)

1. 当用户提交用户名和密码时，UsernamePasswordAuthenticationFilter 将从 HttpServletRequest 中提取用户名和密码，从而创建一个 UsernamePasswordAuthenticationToken，它是一种 Authentication 类型。

2. 接下来，将 UsernamePasswordAuthenticationToken 传递到 AuthenticationManager 进行身份验证。AuthenticationManager 的详细信息取决于用户信息的存储方式。

3. 如果验证失败，则失败。
    + SecurityContextHolder 清除。
    + RememberMeServices.loginFail 被调用。如果没有配置 remember me，这是一个 no-op。
    + AuthenticationFailureHandler 被调用。

4. 如果认证成功，则成功。
    + SessionAuthenticationStrategy 收到新登录的通知。
    + 在 SecurityContextHolder 中设置 Authentication。
    + RememberMeServices.loginSuccess 被调用。如果没有配置 remember me，这是一个 no-op。
    + ApplicationEventPublisher 发布 InteractiveAuthenticationSuccessEvent。
    + 调用 AuthenticationSuccessHandler。通常这是一个 SimpleUrlAuthenticationSuccessHandler，当我们重定向到登录页面时，它会重定向到 ExceptionTranslationFilter 保存的请求。

默认情况下启用了 Spring Security 表单登录。但是，一旦提供了任何基于 servlet 的配置，就必须显式地提供基于表单的登录。一个最小的、显式的 Java 配置可以在下面找到：

```java
protected void configure(HttpSecurity http) {
    http
        // ...
        .formLogin(withDefaults());
}
```

在这个配置中，Spring Security 将呈现一个默认的登录页面。大多数生产应用程序将需要自定义登录表单。

下面的配置演示如何提供自定义登录表单。

```java
protected void configure(HttpSecurity http) throws Exception {
    http
        // ...
        .formLogin(form -> form
            .loginPage("/login")
            .permitAll()
        );
}
```

当在 Spring Security 配置中指定登录页面时，我们将负责呈现页面。下面是一个 Thymeleaf 模板，它生成一个符合 /login 登录页面的 HTML 登录表单。

```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="https://www.thymeleaf.org">
    <head>
        <title>Please Log In</title>
    </head>
    <body>
        <h1>Please Log In</h1>
        <div th:if="${param.error}">Invalid username and password.</div>
        <div th:if="${param.logout}">You have been logged out.</div>
        <form th:action="@{/login}" method="post">
            <div>
                <input type="text" name="username" placeholder="Username"/>
            </div>
            <div>
                <input type="password" name="password" placeholder="Password"/>
            </div>
            <input type="submit" value="Log in" />
        </form>
    </body>
</html>
```

关于默认的 HTML 表单有以下几点要点：
* 表单应该执行 post 到 /login
* 该表单需要包含一个 CSRF 令牌，该令牌由 Thymeleaf 自动包含
* 表单应该在名为 username 的参数中指定用户名
* 表单应该在名为 password 的参数中指定密码
* 如果发现 HTTP 参数错误，则表示用户未能提供有效的用户名/密码
* 如果找到 HTTP 参数 logout，则表示用户已成功注销

许多用户只需要自定义登录页面即可。但是，如果需要，以上所有内容都可以通过附加配置进行定制。

如果正在使用Spring MVC，那么将需要一个将 GET /login 映射到我们创建的登录模板的 controller。LoginController 最小示例如下所示：

```java
@Controller
class LoginController {
    @GetMapping("/login")
    String login() {
        return "login";
    }
}
```

### Basic 身份验证
本节详细介绍 Spring Security 如何为基于 servlet 的应用程序提供对 Basic HTTP 身份验证的支持。

让我们看看 HTTP Basic 身份验证在 Spring Security 中是如何工作的。首先，我们看到 WWW-Authenticate header 被发送回未经身份验证的客户端。

![basicauthenticationentrypoint](/images/springsecurity/basicauthenticationentrypoint.png)

1. 首先，用户向未授权的 resource /private 发出未经身份验证的请求。

2. Spring Security 的 FilterSecurityInterceptor 表示通过抛出 AccessDeniedException 来拒绝未经身份验证的请求。

3. 因为用户没有经过身份验证，ExceptionTranslationFilter 将启动身份验证。配置的 AuthenticationEntryPoint 是一个 BasicAuthenticationEntryPoint 实例，它发送一个 WWW-Authenticate header。RequestCache 通常是一个 NullRequestCache，它不保存请求，因为客户端能够重放它最初请求的请求。

当客户端收到 WWW-Authenticate header 时，它知道应该使用用户名和密码重试。下面是正在处理的用户名和密码流程。

![basicauthenticationentrypoint](/images/springsecurity/basicauthenticationentrypoint.png)

1. 当用户提交用户名和密码时，UsernamePasswordAuthenticationFilter 将从 HttpServletRequest 中提取用户名和密码，从而创建一个 UsernamePasswordAuthenticationToken，它是一种 Authentication 类型。

2. 接下来，将 UsernamePasswordAuthenticationToken 传递到 AuthenticationManager 进行身份验证。AuthenticationManager 的详细信息取决于用户信息的存储方式。

3. 如果验证失败，则失败。
    + SecurityContextHolder 清除。
    + RememberMeServices.loginFail 被调用。如果没有配置 remember me，这是一个 no-op。
    + 调用 AuthenticationEntryPoint 来触发再次发送 WWW-Authenticate。

4. 如果认证成功，则成功。
    + 在 SecurityContextHolder 中设置 Authentication。
    + RememberMeServices.loginSuccess 被调用。如果没有配置 remember me，这是一个 no-op。
    + BasicAuthenticationFilter 调用 FilterChain.doFilter(request,response) 来继续应用程序逻辑的其余部分。

默认情况下，Spring Security 的 HTTP Basic 身份验证支持处于启用状态。但是，一旦提供了任何基于 servlet 的配置，就必须显式地提供 HTTP Basic。

可以在下面找到最小的显式配置：

```java
protected void configure(HttpSecurity http) {
    http
        // ...
        .httpBasic(withDefaults());
}
```

### Digest 身份验证
不应该在现代应用程序中使用 Digest 身份验证，因为它被认为是不安全的。最明显的问题是，必须以明文、加密或 MD5 格式存储密码。所有这些存储格式都被认为是不安全的。相反，应该使用一种单向自适应密码散列（如 bCrypt、PBKDF2、SCrypt 等）来存储凭证，这种方式不受 Digest 身份验证的支持。

### In-Memory 身份验证
Spring Security的 InMemoryUserDetailsManager 实现了UserDetailsService，以支持在内存中检索的基于用户名/密码的身份验证。InMemoryUserDetailsManager 通过实现 UserDetailsManager 接口提供对用户详细信息的管理。当 Spring Security 配置为接受用户名/密码进行身份验证时，它将使用基于 UserDetails 的身份验证。

在本例中，我们使用 Spring Boot CLI 对 password 的密码进行编码，得到的编码密码为 {bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW。

```java
@Bean
public UserDetailsService users() {
    UserDetails user = User.builder()
        .username("user")
        .password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
        .roles("USER")
        .build();
    UserDetails admin = User.builder()
        .username("admin")
        .password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
        .roles("USER", "ADMIN")
        .build();
    return new InMemoryUserDetailsManager(user, admin);
}
```

上面的示例以安全格式存储密码，但是在入门体验方面还有很多需要改进的地方。

在下面的示例中，我们利用 User.withDefaultPasswordEncoder 来确保存储在内存中的密码是受保护的。但是，它不能保护密码不被反编译源代码而获得密码。因此，User.withDefaultPasswordEncoder 应仅用于“入门”，而不用于生产。

```java
@Bean
public UserDetailsService users() {
    // The builder will ensure the passwords are encoded before saving in memory
    UserBuilder users = User.withDefaultPasswordEncoder();
    UserDetails user = users
        .username("user")
        .password("password")
        .roles("USER")
        .build();
    UserDetails user = users
        .username("admin")
        .password("password")
        .roles("USER", "ADMIN")
        .build();
    return new InMemoryUserDetailsManager(user, admin);
}
```

### JDBC 身份验证
Spring Security 的 JdbcDaoImpl 实现了 UserDetailsService，为使用 JDBC 检索的基于用户名/密码的身份验证提供支持。JdbcUserDetailsManager 扩展了 JdbcDaoImpl，通过 UserDetailsManager 接口提供对用户详细信息的管理。当 Spring Security 配置为接受用户名/密码进行身份验证时，它将使用基于 UserDetails 的身份验证。

#### 默认 Schema
Spring Security 为基于 JDBC 的身份验证提供默认查询。本节提供与默认查询一起使用的相应默认 schemas。我们将需要调整此 schema，使其与使用的任何自定义的查询和数据库方言匹配。

##### 用户 Schema
JdbcDaoImpl 要求使用表来加载用户的密码、帐户状态（启用或禁用）和权限（角色）列表。下面可以找到所需的默认 schema。

```sql
create table users
(
    username varchar(50) not null comment '用户名',
    password varchar(100) not null comment '密码',
    enabled  bit(1)      not null comment '是否启用',
    PRIMARY KEY (`username`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;

create table authorities
(
    username  varchar(50) not null comment '用户名',
    authority varchar(255) not null comment '权限'
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;
```

默认 schema 还公开为名为 org/springframework/security/core/userdetails/jdbc/users.ddl 的类路径资源。

##### 用户组 Schema
如果应用程序正在使用用户组，则需要提供用户组 schema。可以在下面找到组的默认 schema。

```sql
create table `groups`
(
    id         bigint AUTO_INCREMENT not null comment '主键',
    group_name varchar(50)           not null comment '用户组名',
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;

create table group_authorities
(
    group_id  bigint      not null comment '用户组id',
    authority varchar(255) not null comment '用户组权限'
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;

create table group_members
(
    id       bigint AUTO_INCREMENT not null comment '主键',
    username varchar(50)           not null comment '用户名',
    group_id bigint                not null comment '用户组id',
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;
```

#### Setting up a DataSource
在配置 JdbcUserDetailsManager 之前，必须创建一个 DataSource。

#### JdbcUserDetailsManager Bean
在本例中，我们使用 Spring Boot CLI 对 password 的密码进行编码，得到的编码密码为 {bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW。

```java
@Bean
UserDetailsManager users(DataSource dataSource) {
    UserDetails user = User.builder()
        .username("user")
        .password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
        .roles("USER")
        .build();
    UserDetails admin = User.builder()
        .username("admin")
        .password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
        .roles("USER", "ADMIN")
        .build();
    JdbcUserDetailsManager users = new JdbcUserDetailsManager(dataSource);
    users.createUser()
}
```

### UserDetails
UserDetails 由 UserDetailsService 返回。DaoAuthenticationProvider 验证 UserDetails，然后返回一个 Authentication，该 Authentication 的主体是由已配置的 UserDetailsService 返回的 UserDetails。

### UserDetailsService
DaoAuthenticationProvider 使用 UserDetailsService 检索用户名、密码和其他用于使用用户名和密码进行身份验证的属性。Spring Security 提供了 UserDetailsService 的内存和 JDBC 实现。

通过将自定义的 UserDetailsService 公开为 bean，可以定义自定义身份验证。例如，假设 CustomUserDetailsService 实现了 UserDetailsService，则以下内容将自定义身份验证：

```java
@Bean
CustomUserDetailsService customUserDetailsService() {
    return new CustomUserDetailsService();
}
```

**注意：** 仅当尚未填充 AuthenticationManagerBuilder 且未定义 AuthenticationProviderBean 时才使用此选项。

### PasswordEncoder
Spring Security 的 servlet 通过与 PasswordEncoder 集成来支持安全存储密码。可以通过公开 PasswordEncoder Bean 来定制 Spring Security 使用的 PasswordEncoder 实现。

### DaoAuthenticationProvider
DaoAuthenticationProvider 是一个 AuthenticationProvider 实现，它利用 UserDetailsService 和 PasswordEncoder 来验证用户名和密码。

让我们来看看 DaoAuthenticationProvider 在 Spring Security 中是如何工作的。该图详细解释了 AuthenticationManager 在读取用户名和密码时的工作方式。

![daoauthenticationprovider](/images/springsecurity/daoauthenticationprovider.png)

1. 读取用户名和密码的身份验证过滤器将 UsernamePasswordAuthenticationToken 传递给 AuthenticationManager，后者由 ProviderManager 实现。

2. ProviderManager 被配置为使用 DaoAuthenticationProvider 类型的 AuthenticationProvider。

3. DaoAuthenticationProvider 从 UserDetailsService 中查找 UserDetails。

4. DaoAuthenticationProvider 使用 PasswordEncoder 验证上一步返回的用户详细信息上的密码。

5. 当身份验证成功时，返回的身份验证是 UsernamePasswordAuthenticationToken 类型，其主体是配置的 UserDetailsService 返回的 UserDetails。最终，身份验证过滤器将在 SecurityContextHolder 中设置返回的 UsernamePasswordAuthenticationToken。

## 会话管理
与 HTTP 会话相关的功能由 SessionManagementFilter 和 SessionAuthenticationStrategy 接口的组合来处理，而 SessionAuthenticationStrategy 接口是由过滤器委托给它的。典型的用法包括会话固定攻击保护、检测会话超时和限制一个经过身份验证的用户可以同时打开多少会话。

### 检测超时
可以配置 Spring Security 来检测无效会话 ID 的提交，并将用户重定向到适当的 URL。这是通过 session-management 元素实现的：

```xml
<http>
    ...
    <session-management invalid-session-url="/invalidSession.htm" />
</http>
```

请注意，如果使用此机制检测会话超时，则如果用户注销然后在不关闭浏览器的情况下重新登录，则可能会错误地报告错误。这是因为当使会话无效时，会话 cookie 未被清除，即使用户已注销，也将重新提交。可以在注销时显式删除 JSESSIONID cookie，例如在注销处理程序中使用以下语法：

```xml
<http>
    <logout delete-cookies="JSESSIONID" />
</http>
```

不幸的是，这不能保证对每个 servlet 容器都有效，因此需要在环境中测试它。

### 并发会话控制
如果希望对单个用户登录应用程序的能力进行限制，Spring Security 通过以下简单的附加功能来提供开箱即用的支持。首先，需要将以下监听器添加到 web.xml 文件中，以保持 Spring Security 更新会话生命周期事件：

```xml
<listener>
    <listener-class>
        org.springframework.security.web.session.HttpSessionEventPublisher
    </listener-class>
</listener>
```

然后将以下行添加到应用程序上下文中：

```xml
<http>
    ...
    <session-management>
        <concurrency-control max-sessions="1" />
    </session-management>
</http>
```

这将防止用户多次登录-第二次登录将导致第一次无效。通常，我们希望防止再次登录，在这种情况下，可以使用

```xml
<http>
    ...
    <session-management>
        <concurrency-control max-sessions="1" error-if-maximum-exceeded="true" />
    </session-management>
</http>
```

第二次登录将被拒绝。“拒绝”是指如果使用基于表单的登录，则用户将被发送到 authentication-failure-url。如果第二次身份验证通过另一个非交互机制（如“remember-me”）进行，则会向客户端发送“unauthorized”（401）错误。如果希望使用错误页面，则可以将属性 session-authentication-error-url 添加到 session-management 元素中。

如果对基于表单的登录使用自定义身份验证过滤器，则必须显式配置并发会话控制支持。

### 会话固定攻击保护
会话固定攻击是一种潜在的风险，恶意攻击者可能通过访问站点来创建会话，然后说服另一个用户使用相同的会话进行登录（例如，通过向他们发送包含会话标识符作为参数的链接）。Spring Security 通过创建新会话或在用户登录时更改会话 ID 来自动防止这种情况。如果不需要这种保护，或者它与其他一些需求相冲突，可以使用 <session management> 上的 session-fixation-protection 属性来控制行为，该属性有四个选项：

* none - 什么都不要做。原始会话将被保留。
* newSession - 创建一个新的“干净”的会话，而不复制现有会话数据（仍将复制与 Spring Security-related 属性）。
* migrateSession - 创建新会话，并将所有现有会话属性复制到新会话。这是 Servlet3.0 或更早版本容器中的默认设置。
* changeSessionId - 不创建新会话。相反，使用 Servlet 容器（HttpServletRequest#changeSessionId()）提供的会话固定保护。此选项仅在 Servlet 3.1 (Java EE 7) 和更新的容器中可用。在旧的容器中指定它将导致异常。这是 Servlet 3.1 和更新的容器中的默认值。

当会话固定保护发生时，它将导致在应用程序上下文中发布 SessionFixationProtectionEvent。如果使用 changeSessionId，此保护还将导致任何 javax.servlet.http.HttpSessionIdListener 将被通知，因此如果代码同时监听这两个事件，需谨慎使用。

###  SessionManagementFilter
SessionManagementFilter 根据 SecurityContextHolder 的当前内容检查 SecurityContextRepository 的内容，以确定用户在当前请求期间是否已通过身份验证，通常是通过非交互式身份验证机制，如 pre-authentication 或 remember-me。如果存储库包含安全上下文，则过滤器不执行任何操作。如果没有，并且线程本地 SecurityContext 包含一个（非匿名）身份验证对象，则过滤器假定它们已由堆栈中的前一个过滤器验证过。然后，它将调用已配置的 SessionAuthenticationStrategy。

如果用户当前未通过身份验证，则过滤器将检查是否请求了无效的会话 ID（例如，由于超时），并将调用配置的 InvalidSessionStrategy（如果已设置）。最常见的行为只是重定向到一个固定的 URL，它被封装在标准实现 SimpleRedirectInvalidSessionStrategy 中。如前所述，在通过命名空间配置无效会话 URL 时也使用后者。

### SessionAuthenticationStrategy
SessionAuthenticationStrategy 由 SessionManagementFilter 和 AbstractAuthenticationProcessingFilter 共同使用，因此，如果使用自定义的表单登录类，例如，需要将它注入到这两个类中。在本例中，将命名空间和自定义 beans 组合在一起的典型配置可能如下所示：

```xml
<project>
    <http>
        <custom-filter position="FORM_LOGIN_FILTER" ref="myAuthFilter" />
        <session-management session-authentication-strategy-ref="sas"/>
    </http>
    
    <beans:bean id="myAuthFilter" class="org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter">
        <beans:property name="sessionAuthenticationStrategy" ref="sas" />
        ...
    </beans:bean>
    
    <beans:bean id="sas" class="org.springframework.security.web.authentication.session.SessionFixationProtectionStrategy" />
</project>
```

注意，如果在实现 HttpSessionBindingListener 的会话中存储 beans(包括 Spring session-scoped beans)，则使用默认的 SessionFixationProtectionStrategy 可能会导致问题。

### 并发控制
Spring Security 能够防止主体对同一个应用程序同时进行身份验证的次数超过指定的次数。许多 ISVs 利用这一点来加强授权，而网络管理员喜欢这个特性，因为它有助于防止人们共享登录名。例如，我们可以阻止用户“Batman”从两个不同的会话登录到 web 应用程序。我们可以终止他们以前的登录，也可以在他们再次尝试登录时报告错误，以阻止第二次登录。注意，如果使用第二种方法，未显式注销（例如，刚刚关闭浏览器的用户）的用户将无法再次登录，直到其原始会话过期。

命名空间支持并发控制。

该实现使用 SessionAuthenticationStrategy 的专用版本，称为 ConcurrentSessionControlAuthenticationStrategy。

要启用并发会话支持，需要将以下内容添加到 web.xml：

```xml
<listener>
    <listener-class>
        org.springframework.security.web.session.HttpSessionEventPublisher
    </listener-class>
</listener>
```

此外，还需要将 ConcurrentSessionFilter 添加到 FilterChainProxy 中。ConcurrentSessionFilter 
需要两个构造函数参数 sessionRegistry(通常指向 SessionRegistryImpl 的实例) 和 sessionInformationExpiredStrategy(定义会话过期时要应用的策略)。
使用命名空间创建 FilterChainProxy 和其他默认 beans 的配置可能如下所示：

```xml
<project>
    <http>
        <custom-filter position="CONCURRENT_SESSION_FILTER" ref="concurrencyFilter" />
        <custom-filter position="FORM_LOGIN_FILTER" ref="myAuthFilter" />
        
        <session-management session-authentication-strategy-ref="sas"/>
    </http>
    
    <beans:bean id="redirectSessionInformationExpiredStrategy" class="org.springframework.security.web.session.SimpleRedirectSessionInformationExpiredStrategy">
        <beans:constructor-arg name="invalidSessionUrl" value="/session-expired.htm" />
    </beans:bean>
        
    <beans:bean id="concurrencyFilter" class="org.springframework.security.web.session.ConcurrentSessionFilter">
        <beans:constructor-arg name="sessionRegistry" ref="sessionRegistry" />
        <beans:constructor-arg name="sessionInformationExpiredStrategy" ref="redirectSessionInformationExpiredStrategy" />
    </beans:bean>
        
    <beans:bean id="myAuthFilter" class="org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter">
        <beans:property name="sessionAuthenticationStrategy" ref="sas" />
        <beans:property name="authenticationManager" ref="authenticationManager" />
    </beans:bean>
    
    <beans:bean id="sas" class="org.springframework.security.web.authentication.session.CompositeSessionAuthenticationStrategy">
        <beans:constructor-arg>
            <beans:list>
                <beans:bean class="org.springframework.security.web.authentication.session.ConcurrentSessionControlAuthenticationStrategy">
                    <beans:constructor-arg ref="sessionRegistry"/>
                    <beans:property name="maximumSessions" value="1" />
                    <beans:property name="exceptionIfMaximumExceeded" value="true" />
                </beans:bean>
                <beans:bean class="org.springframework.security.web.authentication.session.SessionFixationProtectionStrategy">
                </beans:bean>
                <beans:bean class="org.springframework.security.web.authentication.session.RegisterSessionAuthenticationStrategy">
                    <beans:constructor-arg ref="sessionRegistry"/>
                </beans:bean>
            </beans:list>
        </beans:constructor-arg>
    </beans:bean>
    
    <beans:bean id="sessionRegistry" class="org.springframework.security.core.session.SessionRegistryImpl" />
</project>
```

将监听器添加到 web.xml 会导致每次 HttpSession 开始或终止时，ApplicationEvent 都被发布到 Spring ApplicationContext。这是至关重要的，因为它允许在会话结束时通知 SessionRegistryImpl。没有它，即使用户退出了另一个会话或会话超时，一旦他们超出了会话限额，也将无法再次登录。

#### 在 SessionRegistry 中查询当前经过身份验证的用户及其会话
通过命名空间或使用纯bean设置并发控制有一个有用的副作用，即为您提供对SessionRegistry的引用，您可以直接在应用程序中使用该引用，因此即使您不想限制用户可能拥有的会话数，无论如何，建立基础设施可能是值得的。您可以将maximumSession属性设置为-1以允许无限会话。如果您使用的是命名空间，那么可以使用session registry alias属性为内部创建的SessionRegistry设置别名，提供一个可以注入到您自己的bean中的引用。

getAllPrincipals() 方法提供当前已验证用户的列表。我们可以通过调用 getAllSessions(Object principal, boolean includeExpiredSessions) 方法来列出用户的会话，该方法返回 SessionInformation 对象的列表。还可以通过在 SessionInformation 实例上调用 expireNow() 来终止用户的会话。当用户返回到应用程序时，将阻止他们继续操作。例如，我们可能会发现这些方法在管理应用程序中非常有用。

## Remember-Me 身份验证
### 概述
Remember-me 或 persistent-login 身份验证是指网站能够记住会话之间主体的身份。这通常是通过将 cookie 发送到浏览器来完成的，cookie 将在以后的会话中被检测，并导致自动登录。Spring Security 为这些操作提供了必要的钩子，并且有两个具体的 remember-me 实现。一种使用散列来保护基于 cookie 的令牌的安全性，另一种使用数据库或其他持久存储机制来存储生成的令牌。

注意，这两个实现都需要一个 UserDetailsService。如果我们使用的身份验证提供程序不使用 UserDetailsService（例如，LDAP提供程序），那么除非应用程序上下文中也有 UserDetailsService bean，否则它将无法工作。

### 基于简单哈希的令牌方法
这种方法使用哈希来实现有用的“remember-me”策略。本质上，cookie 在成功的交互身份验证后发送到浏览器，cookie 的组成如下：

```text
base64(username + ":" + expirationTime + ":" +
md5Hex(username + ":" + expirationTime + ":" password + ":" + key))

username:          As identifiable to the UserDetailsService
password:          That matches the one in the retrieved UserDetails
expirationTime:    The date and time when the remember-me token expires, expressed in milliseconds
key:               A private key to prevent modification of the remember-me token
```

因此，remember-me 令牌仅在指定的时间段内有效，前提是用户名、密码和密钥不会更改。值得注意的是，这有一个潜在的安全问题，因为在令牌过期之前，从任何用户代理捕获的 remember-me 令牌都是可用的。这与 digest 身份验证的问题相同。如果主体知道某个令牌已被捕获，则可以轻松更改其密码，并在出现问题时立即使所有“remember-me”令牌失效。如果需要更重要的安全性，则应使用下一节中描述的方法。或者，remember-me 服务根本不应该使用。

如果我们熟悉名称空间配置，只需添加 <remember-me> 元素即可启用 remember-me 身份验证：

```xml
<http>
    ...
    <remember-me key="myAppKey"/>
</http>
```

通常会自动选择 UserDetailsService。如果应用程序上下文中有多个，则需要指定哪个应该与 user-service-ref 属性一起使用，其值是 UserDetailsService bean 的名称。

### 持久令牌方法
这种方法基于[文章](http://jaspan.com/improved_persistent_login_cookie_best_practice)，并做了一些小的修改。若要将此方法用于命名空间配置，需要提供一个数据源引用：

```xml
<http>
    ...
    <remember-me data-source-ref="someDataSource"/>
</http>
```

数据库应包含使用以下 SQL（或等效 SQL）创建的持久登录表：

```sql
create table persistent_logins (username varchar(64) not null,
                                series varchar(64) primary key,
                                token varchar(64) not null,
                                last_used timestamp not null)
```

### Remember-Me 接口和实现
Remember-Me 与 UsernamePasswordAuthenticationFilter 一起使用，并通过 AbstractAuthenticationProcessingFilter 超类中的钩子实现。它也用于 BasicAuthenticationFilter 中。钩子将在适当的时间调用具体的 RememberMeServices。接口如下所示：

```java
Authentication autoLogin(HttpServletRequest request, HttpServletResponse response);

void loginFail(HttpServletRequest request, HttpServletResponse response);

void loginSuccess(HttpServletRequest request, HttpServletResponse response, Authentication successfulAuthentication);
```

注意，在这个阶段，AbstractAuthenticationProcessingFilter 只调用 loginFail() 和 loginSuccess() 方法。每当 SecurityContextHolder 不包含身份验证时，RememberAuthenticationFilter 就会调用 autoLogin() 方法。因此，这个接口为底层的 remember-me 实现提供与身份验证相关事件的充分通知，并在候选 web 请求可能包含 cookie 并希望被记住时将其委托给实现。这种设计允许任意数量的“remember-me”实现策略。我们在上面已经看到，Spring Security 提供了两个实现。我们将依次研究这些。

#### TokenBasedRememberMeServices
这个实现支持基于简单哈希的令牌方法中描述的简单方法。TokenBasedRememberMeServices 生成 RememberAuthenticationToken，由 RememberAuthenticationProvider 处理。此身份验证提供程序和 TokenBasedRememberMeServices 之间共享密钥。此外，TokenBasedRememberMeServices 需要一个 UserDetailsService，它可以从中检索用户名和密码以进行签名比较，并生成 RememberAuthenticationToken 以包含正确的 GrantedAuthority。应用程序应该提供某种注销命令，如果用户请求，该命令将使 cookie 无效。TokenBasedRememberMeServices 还实现了 Spring Security 的 LogoutHandler 接口，因此可以与LogoutFilter一起使用以自动清除 cookie。

应用程序上下文中启用 remember-me 服务所需的 beans 如下：

```xml
<project>
    <bean id="rememberMeFilter" class="org.springframework.security.web.authentication.rememberme.RememberMeAuthenticationFilter">
        <property name="rememberMeServices" ref="rememberMeServices"/>
        <property name="authenticationManager" ref="theAuthenticationManager" />
    </bean>
    
    <bean id="rememberMeServices" class="org.springframework.security.web.authentication.rememberme.TokenBasedRememberMeServices">
        <property name="userDetailsService" ref="myUserDetailsService"/>
        <property name="key" value="springRocks"/>
    </bean>
    
    <bean id="rememberMeAuthenticationProvider" class="org.springframework.security.authentication.RememberMeAuthenticationProvider">
        <property name="key" value="springRocks"/>
    </bean>
</project>
```

不要忘记将 RememberMeServices 实现添加到 UsernamePasswordAuthenticationFilter.setRememberMeServices() 属性中，包括在 AuthenticationManager.setProviders() 列表中添加 RememberMeAuthenticationProvider，并将 RememberMeAuthenticationFilter 添加到 FilterChainProxy 中（通常紧跟在 UsernamePasswordAuthenticationFilter 之后）。

#### PersistentTokenBasedRememberMeServices
这个类可以与 TokenBasedRememberMeServices 以相同的方式使用，但是它还需要配置一个 PersistentTokenRepository 来存储令牌。有两种标准实现。

* InMemoryTokenRepositoryImpl 仅用于测试。

* JdbcTokenRepositoryImpl 它将令牌存储在数据库中。

上面用持久令牌方法描述了数据库模式。

## 匿名身份验证
### 概述
采用“deny-by-default”通常被认为是一种良好的安全实践，在这种情况下，可以明确指定允许的内容，而不允许其他任何内容。定义未经身份验证的用户可以访问的内容也是类似的情况，特别是对于 web 应用程序。许多站点要求用户必须对除少数 URLs(例如主页和登录页面)以外的任何内容进行身份验证。在这种情况下，为这些特定的 URLs 定义访问配置属性比为每个受保护的资源定义访问配置属性更容易。换句话说，有时在默认情况下，ROLE_SOMETHING 是必需的，并且只允许此规则的某些例外情况，比如应用程序的登录、注销和主页。也可以完全从过滤器链中忽略这些页面，从而绕过访问控制检查，但是由于其他原因，这可能是不可取的，特别是对于经过身份验证的用户，页面的行为可能有所不同。

这就是我们所说的匿名身份验证。注意，“anonymously authenticated"”的用户和未经身份验证的用户之间并没有真正的概念上的区别。Spring Security 的匿名身份验证为我们配置访问控制属性提供了更方便的方法。例如，对诸如 getCallerPrincipal 之类的 servlet API 的调用仍将返回 null，即使 SecurityContextHolder 中实际上有一个匿名身份验证对象。

在其他情况下，匿名身份验证是有用的，例如当审计拦截器查询 SecurityContextHolder 以确定哪个主体负责给定的操作时。如果类知道 SecurityContextHolder 始终包含一个 Authentication 对象，并且从不为 null，那么它们可以更健壮地编写。

### 配置
使用 HTTP 配置 Spring Security 3.0 时，会自动提供匿名身份验证支持，并且可以使用 <Anonymous> 元素自定义（或禁用）。除非使用传统的 bean 配置，否则不需要配置这里描述的 beans。

三个类一起提供匿名身份验证功能。AnonymousAuthenticationToken 是 Authentication 的一种实现，它存储应用于匿名主体的授权。
有一个对应的 AnonymousAuthenticationProvider，它被链接到 ProviderManager中，以便接受 AnonymousAuthenticationToken。
最后，还有一个 AnonymousAuthenticationFilter，它在正常的身份验证机制之后被链接起来，如果没有现有的 Authentication，
它会自动将 AnonymousAuthenticationToken 添加到 SecurityContextHolder。过滤器和身份验证提供程序的定义如下所示：

```xml
<project>
    <bean id="anonymousAuthFilter" class="org.springframework.security.web.authentication.AnonymousAuthenticationFilter">
        <property name="key" value="foobar"/>
        <property name="userAttribute" value="anonymousUser,ROLE_ANONYMOUS"/>
    </bean>
    
    <bean id="anonymousAuthenticationProvider" class="org.springframework.security.authentication.AnonymousAuthenticationProvider">
        <property name="key" value="foobar"/>
    </bean>
</project>
```

密钥在过滤器和身份验证提供程序之间共享，以便前者创建的令牌被后者接受。userAttribute 以 usernameInTheAuthenticationToken、grantedAuthority 的形式表示。这与 InMemoryDaoImpl 的 userMap 属性的等号后面使用的语法相同。

如前所述，匿名身份验证的好处是，所有 URI 模式都可以对其应用安全性。例如：

```xml
<bean id="filterSecurityInterceptor" class="org.springframework.security.web.access.intercept.FilterSecurityInterceptor">
    <property name="authenticationManager" ref="authenticationManager"/>
    <property name="accessDecisionManager" ref="httpRequestAccessDecisionManager"/>
    <property name="securityMetadata">
        <security:filter-security-metadata-source>
        <security:intercept-url pattern='/index.jsp' access='ROLE_ANONYMOUS,ROLE_USER'/>
        <security:intercept-url pattern='/hello.htm' access='ROLE_ANONYMOUS,ROLE_USER'/>
        <security:intercept-url pattern='/logoff.jsp' access='ROLE_ANONYMOUS,ROLE_USER'/>
        <security:intercept-url pattern='/login.jsp' access='ROLE_ANONYMOUS,ROLE_USER'/>
        <security:intercept-url pattern='/**' access='ROLE_USER'/>
        </security:filter-security-metadata-source>" +
    </property>
</bean>
```

### AuthenticationTrustResolver
完成匿名身份验证讨论的是 AuthenticationTrustResolver 接口及其相应的 AuthenticationTrustResolverImpl 实现。这个接口提供了一个 isAnonymous（身份验证）方法，它允许感兴趣的类考虑这种特殊类型的身份验证状态。ExceptionTranslationFilter 在处理 AccessDeniedException 时使用此接口。如果引发 AccessDeniedException，并且身份验证是匿名类型，则过滤器将启动 AuthenticationEntryPoint，以便主体能够正确进行身份验证，而不是引发403（禁止）响应。这是一个必要的区别，否则主体将始终被视为“经过身份验证”，并且永远不会有机会通过表单、basic、digest 或其他一些常规身份验证机制登录。

我们经常会看到上述拦截器配置中的 ROLE_ANONYMOUS 属性被替换为 IS_AUTHENTICATED_ANONYMOUSLY，这实际上与定义访问控制时的情况相同。这是一个使用 AuthenticatedVoter 的示例，我们将在授权一章中看到。它使用 AuthenticationTrustResolver 来处理这个特定的配置属性并将访问权授予匿名用户。AuthenticatedVoter 方式更强大，因为它允许我们区分 anonymous、remember-me 和 fully-authenticated 的用户。如果我们不需要这个功能，那么可以使用ROLE_ANONYMOUS，它将由 Spring Security 的标准 RoleVoter 处理。

## 预认证场景
在某些情况下，我们希望使用 Spring Security 进行授权，但是用户在访问应用程序之前已经通过了某些外部系统的可靠身份验证。我们将这些情况称为“预认证”场景。示例包括 X.509、Siteminder 和运行应用程序的 Java EE 容器的身份验证。当使用预认证时，Spring Security 必须

* 确定提出请求的用户。
* 获取用户的权限。

详细信息将取决于外部身份验证机制。对于 X.509，用户可以通过其证书信息进行标识，对于 Siteminder，可以通过 HTTP 请求头进行标识。如果依赖容器身份验证，则将通过对传入的 HTTP 请求调用 getUserPrincipal() 方法来标识用户。在某些情况下，外部机制可能为用户提供角色/权限信息，但在其他情况下，必须从单独的源（如 UserDetailsService）获取权限。

### 预认证框架类
因为大多数预认证机制都遵循相同的模式，所以 Spring Security 有一组类，它们为实现预认证认证提供者提供了一个内部框架。这消除了重复，并允许以结构化的方式添加新的实现，而不必从头开始编写所有内容。如果我们想使用像 X.509 身份验证这样的东西，我们不需要知道这些类，因为它已经有一个名称空间配置选项，使用起来更简单。如果我们需要使用显式 bean 配置，或者正在计划编写自己的实现，那么了解所提供的实现是如何工作的将非常有用。我们将在 org.springframework.security.web.authentication.preauth 包下找到类。我们只是在这里提供一个概要。

#### AbstractPreAuthenticatedProcessingFilter
这个类将检查安全上下文的当前内容，如果为空，则将尝试从 HTTP 请求中提取用户信息并将其提交给 AuthenticationManager。子类重写以下方法以获取此信息：

```java
protected abstract Object getPreAuthenticatedPrincipal(HttpServletRequest request);

protected abstract Object getPreAuthenticatedCredentials(HttpServletRequest request);
```

调用这些之后，过滤器将创建一个包含返回数据的 PreAuthenticatedAuthenticationToken，并将其提交进行身份验证。这里的“authentication”实际上是指进一步的处理，以加载用户的权限，但是遵循了标准的 Spring Security 身份验证体系结构。

与其他 Spring Security 身份验证过滤器一样，预身份验证过滤器具有 authenticationDetailsSource  属性，默认情况下，该属性将创建一个 WebAuthenticationDetails 对象来存储更多的信息，比如会话标识符和原始IP地址等附加信息。如果用户角色信息可以从预认证机制中获得，则数据也存储在此属性中，详细信息实现 GrantedAuthoritiesContainer 接口。这使身份验证提供者能够读取外部分配给用户的权限。接下来我们会看一个具体的例子。

##### J2eeBasedPreAuthenticatedWebAuthenticationDetailsSource
如果过滤器配置了 authenticationDetailsSource，而 authenticationDetailsSource 是此类的实例，则通过为每个预定义的“mappable roles”集合调用  isUserInRole(String role) 方法来获取权限信息。类从已配置的 MappableAttributesRetriever 获取这些。可能的实现包括在应用程序上下文中硬编码列表和从 web.xml 文件中的 <security-role> 信息读取角色信息。预认证示例应用程序使用后一种方法。

还有一个额外的阶段，角色（或属性）使用配置的 Attributes2GrantedAuthoritiesMapper 映射到 Spring Security GrantedAuthority 对象。默认设置只是在名称中添加普通的 ROLE_ 前缀，但它可以让我们完全控制行为。

#### PreAuthenticatedAuthenticationProvider
预验证的提供程序只需为用户加载 UserDetails 对象。它通过委托给 AuthenticationUserDetailsService 来实现这一点。后者类似于标准的 UserDetailsService，但采用 Authentication 对象而不仅仅是用户名：

```java
public interface AuthenticationUserDetailsService {
    UserDetails loadUserDetails(Authentication token) throws UsernameNotFoundException;
}
```

这个接口可能还有其他用途，但是通过预身份验证，它允许访问打包在 Authentication 对象中的权限，正如我们在上一节中看到的那样。PreAuthenticatedGrantedAuthoritiesUserDetailsService 类执行此操作。或者，它可以通过 UserDetailsByNameServiceWrapper 实现委托给标准的 UserDetailsService。

#### Http403ForbiddenEntryPoint
AuthenticationEntryPoint 负责启动未经身份验证的用户的身份验证过程（当他们试图访问受保护的资源时），但在预身份验证的情况下不适用。如果未将预身份验证与其他身份验证机制结合使用，则只能使用此类的实例配置 ExceptionTranslationFilter。如果用户被 AbstractPreAuthenticatedProcessingFilter 拒绝，导致身份验证为 null，则将调用此方法。如果调用，它总是返回 403 禁止的响应码。

### 具体实现
X.509 身份验证在其自己的章节中有介绍。在这里，我们将研究一些类，它们为其他预认证场景提供支持。

#### 请求头身份验证（Siteminder）
外部身份验证系统可以通过在 HTTP 请求上设置特定的头向应用程序提供信息。一个著名的例子是 Siteminder，它在一个名为 SM_USER 的 header 中传递用户名。类 RequestHeaderAuthenticationFilter 支持这种机制，它只是从 header 中提取用户名。默认使用名称 SM_USER 作为标题名。

**注意：** 当使用这样的系统时，框架根本不执行任何身份验证检查，正确配置外部系统并保护对应用程序的所有访问是非常重要的。如果攻击者能够在其原始请求中伪造 headers 而不被检测到，那么他们可以选择任何他们想要的用户名。

##### Siteminder 示例配置
使用此过滤器的典型配置如下所示：

```xml
<project>
    <security:http>
        <!-- Additional http configuration omitted -->
        <security:custom-filter position="PRE_AUTH_FILTER" ref="siteminderFilter" />
    </security:http>
    
    <bean id="siteminderFilter" class="org.springframework.security.web.authentication.preauth.RequestHeaderAuthenticationFilter">
        <property name="principalRequestHeader" value="SM_USER"/>
        <property name="authenticationManager" ref="authenticationManager" />
    </bean>
    
    <bean id="preauthAuthProvider" class="org.springframework.security.web.authentication.preauth.PreAuthenticatedAuthenticationProvider">
        <property name="preAuthenticatedUserDetailsService">
            <bean id="userDetailsServiceWrapper" class="org.springframework.security.core.userdetails.UserDetailsByNameServiceWrapper">
                <property name="userDetailsService" ref="userDetailsService"/>
            </bean>
        </property>
    </bean>
    
    <security:authentication-manager alias="authenticationManager">
        <security:authentication-provider ref="preauthAuthProvider" />
    </security:authentication-manager>
</project>
```

我们在这里假设安全命名空间用于配置。还假设已经向配置中添加了一个 UserDetailsService（称为“userDetailsService”），以加载用户的角色。

#### Java EE 容器身份验证
类 J2eePreAuthenticatedProcessingFilter 将从 HttpServletRequest的userPrincipal 属性中提取用户名。这个过滤器的使用通常与前面在 J2eeBasedPreAuthenticatedWebAuthenticationDetailsSource 中描述的 Java EE 角色的使用相结合。

代码库中有一个使用这种方法的示例应用程序，因此，如果感兴趣，可以从 github 获取代码并查看应用程序上下文文件。代码在 samples/xml/preauth 目录中。

## Run-As 身份验证替换
### 概述
AbstractSecurityInterceptor 能够在安全对象回调阶段临时替换 SecurityContext 和 SecurityContextHolder 中的身份验证对象。只有当 AuthenticationManager 和 AccessDecisionManager 成功地处理了原始身份验证对象时，才会发生这种情况。RunAsManager 将指示应该在 SecurityInterceptorCallback 期间使用的替换 Authentication 对象(如果有的话)。

通过在安全对象回调阶段临时替换 Authentication 对象，受保护的调用将能够调用需要不同身份验证和授权凭据的其他对象。它还可以对特定的 GrantedAuthority 执行任何内部安全检查。由于 Spring Security 提供了大量的帮助类，这些帮助类根据 SecurityContextHolder 的内容自动配置远程处理协议，因此在调用远程 web 服务时，这些作为 run-as 替换特别有用。

### 配置
Spring Security 提供了 RunAsManager 接口：

```jav
Authentication buildRunAs(Authentication authentication, Object object, List<ConfigAttribute> config);

boolean supports(ConfigAttribute attribute);

boolean supports(Class clazz);
```

第一个方法返回 Authentication 对象，该对象应该在方法调用期间替换现有的 Authentication 对象。如果该方法返回 null，则表示不应进行替换。AbstractSecurityInterceptor 使用第二个方法作为其配置属性启动验证的一部分。安全拦截器实现调用 supports(Class) 方法，以确保配置的RunAsManager支持安全侦听器将提供的安全对象类型。

Spring Security 提供了 RunAsManager 的一个具体实现。如果任何 ConfigAttribute 以 RUN_AS_ 开头，则 RunAsManagerImpl 类返回替换的 RunAsUserToken。如果找到任何这样的 ConfigAttribute，则替换的 RunAsUserToken 将包含与原始 Authentication 对象相同的主体、凭据和授予的权限，并为每个 RUN_AS_ ConfigAttribute 提供一个新的 SimpleGrantedAuthority。每个新的 SimpleGrantedAuthority 都将以 ROLE_ 作为前缀，然后是 RUN_AS ConfigAttribute。例如，RUN_AS_SERVER 将导致替换 RunAsUserToken，其中包含授予的 ROLE_RUN_AS_SERVER 权限。

替换的 RunAsUserToken 与任何其他 Authentication 对象一样。它需要通过 AuthenticationManager 进行身份验证，可能需要将其委托给合适的 AuthenticationProvider。RunAsImplAuthenticationProvider 执行这种身份验证。它只是接受，任何 RunAsUserToken 都是有效的。

为了确保恶意代码不会创建 RunAsUserToken 并将其提供给 RunAsImplAuthenticationProvider，密钥的哈希将存储在所有生成的令牌中。RunAsManagerImpl 和 RunAsImplAuthenticationProvider 是在 bean 上下文中使用相同的密钥创建的：

```xml
<bean id="runAsManager" class="org.springframework.security.access.intercept.RunAsManagerImpl">
    <property name="key" value="my_run_as_password"/>
</bean>

<bean id="runAsAuthenticationProvider" class="org.springframework.security.access.intercept.RunAsImplAuthenticationProvider">
    <property name="key" value="my_run_as_password"/>
</bean>
```

通过使用相同的密钥，可以验证每个 RunAsUserToken 是否由已批准的 RunAsManagerImpl 创建。出于安全原因，RunAsUserToken 在创建后是不可变的。

## 处理注销
### 注销 Java 配置
当使用 WebSecurityConfigurerAdapter 时，会自动应用注销功能。默认情况下，访问 URL /logout 将通过以下方式将用户注销：

* 使 HTTP Session 无效
* 清除任何已配置的 RememberMe 身份验证
* 清理 SecurityContextHolder
* 重定向到 /login?logout

然而，与配置登录功能类似，我们也有各种选项来进一步定制的注销要求：

```java
protected void configure(HttpSecurity http) throws Exception {
    http
        .logout(logout -> logout                                                
            .logoutUrl("/my/logout")                                            
            .logoutSuccessUrl("/my/index")                                      
            .logoutSuccessHandler(logoutSuccessHandler)                         
            .invalidateHttpSession(true)                                        
            .addLogoutHandler(logoutHandler)                                    
            .deleteCookies(cookieNamesToClear)                                  
        )
        ...
}
```

1. 提供注销的支持。使用 WebSecurityConfigurerAdapter 时会自动应用此选项。

2. 触发登出的 URL(默认为 /logout)。如果启用了 CSRF 保护（默认），那么请求也必须是POST。

3. 注销后要重定向到的 URL。默认是 /login?logout。

4. 让我们指定一个自定义 LogoutSuccessHandler。如果指定了这个参数，logoutSuccessUrl() 将被忽略。

5. 指定在注销时是否使 HttpSession 无效。默认情况下这是正确的。在幕后配置 SecurityContextLogoutHandler。

6. 添加一个 LogoutHandler。SecurityContextLogoutHandler 被默认添加为最后一个 LogoutHandler。

7. 允许指定要在注销成功时删除的 cookie 的名称。这是显式添加 CookieClearingLogoutHandler 的快捷方式。

通常，为了定制注销功能，我们可以添加 LogoutHandler 和/或 LogoutSuccessHandler 实现。对于许多常见的场景，这些处理程序在使用 fluent API 时是在后台应用的。

### LogoutHandler
通常，LogoutHandler 实现表示能够参与注销处理的类。它们将被调用来执行必要的清理。因此，它们不应该抛出异常。提供了各种实现：

* PersistentTokenBasedRememberMeServices
* TokenBasedRememberMeServices
* CookieClearingLogoutHandler
* CsrfLogoutHandler
* SecurityContextLogoutHandler
* HeaderWriterLogoutHandler

fluent API 并没有直接提供 LogoutHandler 实现，而是提供了快捷方式，在幕后提供了相应的 LogoutHandler 实现。例如，deleteCookies() 允许指定注销成功时要删除的一个或多个 Cookie 的名称。与添加 CookieClearingLogoAuthandler 相比，这是一个快捷方式。

### LogoutSuccessHandler
LogoutSuccessHandler 在 LogoutFilter 成功注销后调用，以处理重定向或转发到适当的目标。请注意，接口几乎与 LogoutHandler 相同，但可能会引发异常。

提供了以下实现：

* SimpleUrlLogoutSuccessHandler 
* HttpStatusReturningLogoutSuccessHandler

如上所述，不需要直接指定 SimpleUrlLogoutSuccessHandler 。相反，fluent API 通过设置 logoutSuccessUrl() 提供了一个快捷方式。将在幕后设置 SimpleUrlLogoutSuccessHandler。所提供的 URL 将在注销后重定向到。默认值是 /login?logout。

HttpStatusReturningLogoutSuccessHandler 在 REST API 类型的场景中可能很有趣。此 LogoutSuccessHandler 允许我们提供要返回的纯 HTTP 状态代码，而不是在成功注销时重定向到 URL。如果未配置，默认情况下将返回状态代码 200。

## 参考资料
1. [Authentication](https://docs.spring.io/spring-security/site/docs/5.3.0.RELEASE/reference/html5/#servlet-authentication)