---
title: Spring Security 参考文档-授权
date: 2020-03-07 11:50:00
categories: Spring Security
tags:
    - Spring Security
---
Spring Security 中的高级授权功能是其流行的最引人注目的原因之一。无论选择如何进行身份验证—无论是使用 Spring Security-provided 机制和提供者，还是与容器或其他非 Spring Security 身份验证授权机构集成—都会发现授权服务可以在应用程序中以一致且简单的方式使用。

在本部分中，我们将探讨不同的 AbstractSecurityInterceptor 实现，然后我们将继续研究如何通过使用域访问控制列表来调整授权。

## 授权架构
### 权限
Authentication，讨论所有 Authentication 实现如何存储 GrantedAuthority 对象列表。这些代表了授予主体的权力。GrantedAuthority 由 AuthenticationManager 插入到 Authentication 对象中，然后由 AccessDecisionManager 在作出授权决策时读取。

GrantedAuthority 是一个只有一个方法的接口：

```java
String getAuthority();
```

此方法允许 AccessDecisionManager 获取 GrantedAuthority 的精确字符串表示形式。通过返回一个表示为字符串的形式，大多数 AccessDecisionManager 可以轻松地“读取” GrantedAuthority。如果 GrantedAuthority 不能精确地表示为字符串，则认为 GrantedAuthority 是“复杂的”，getAuthority() 必须返回 null。

“复杂” GrantedAuthority 的一个例子是一个实现，它存储一个操作列表和适用于不同客户帐号的权限阈值。将这个复杂的 GrantedAuthority 表示为字符串非常困难，因此 getAuthority() 方法应该返回 null。这将向任何 AccessDecisionManager 指示，它将需要专门支持 GrantedAuthority 实现，以便理解其内容。

Spring Security 包括一个具体的 GrantedAuthority 实现 SimpleGrantedAuthority。这允许将任何用户指定的字符串转换为 GrantedAuthority。安全体系结构中包含的所有 AuthenticationProviders 都使用 SimpleGrantedAuthority 来填充 Authentication 对象。

### 预调用处理
Spring Security 提供了拦截器，用于控制对安全对象（如方法调用或 web 请求）的访问。AccessDecisionManager将在调用前决定是否允许继续调用。

#### AccessDecisionManager
AccessDecisionManager 由 AbstractSecurityInterceptor 调用，负责做出最终的访问控制决策。AccessDecisionManager 接口包含三个方法：

```java
void decide(Authentication authentication， Object secureObject，
    Collection<ConfigAttribute> attrs) throws AccessDeniedException;

boolean supports(ConfigAttribute attribute);

boolean supports(Class clazz);
```

AccessDecisionManager 的 decide 方法将传递它所需的所有相关信息，以便进行授权决策。特别是，传递安全对象可以检查实际安全对象调用中包含的那些参数。例如，假设安全对象是 MethodInvocation。查询 MethodInvocation 中的任何客户参数都很容易，然后在 AccessDecisionManager 中实现某种安全逻辑，以确保允许主体对该客户进行操作。如果访问被拒绝，则实现将抛出 AccessDeniedException。

AbstractSecurityInterceptor 在启动时调用 supports(ConfigAttribute) 方法，以确定 AccessDecisionManager 是否可以处理传递的 ConfigAttribute。安全拦截器实现调用 supports(Class) 方法，以确保配置的 AccessDecisionManager 支持安全拦截器将提供的安全对象的类型。

#### 基于投票的 AccessDecisionManager 实现
虽然用户可以实现自己的 AccessDecisionManager 来控制授权的所有方面，但 Spring Security 包含几个基于投票的 AccessDecisionManager 实现。投票决策管理器演示了相关的类。

![access-decision-voting](/images/springsecurity/access-decision-voting.png)

使用此方法，将对一系列 AccessDecisionVoter 实现进行授权决策的轮询。然后 AccessDecisionManager 根据它对投票的评估来决定是否抛出 AccessDeniedException。

AccessDecisionVoter 接口有三个方法：

```java
int vote(Authentication authentication， Object object， Collection<ConfigAttribute> attrs);

boolean supports(ConfigAttribute attribute);

boolean supports(Class clazz);
```

具体的实现返回一个 int，可能的值反映在 AccessDecisionVoter 静态字段 ACCESS_ABSTAIN、ACCESS_DENIED 和 ACCESS_GRANTED 中。如果投票实现对授权决策没有意见，则返回 ACCESS_ABSTAIN。如果它有意见，它必须返回 ACCESS_DENIED 或 ACCESS_GRANTED。

Spring Security 提供了三个具体的 AccessDecisionManager 来记录投票。ConsensusBased 实现将基于不弃权投票的共识批准或拒绝访问。属性用于在投票相等或所有投票弃权的情况下控制行为。如果收到一个或多个 ACCESS_GRANTED 投票，AffirmativeBased 将授予访问权（即，如果至少有一个授予投票，则拒绝投票将被忽略）。与 ConsensusBased 一样，如果所有选民都弃权，则有一个参数控制行为。UnanimousBased 提供者期望一致的 ACCESS_GRANTED 投票来授予访问权限，而忽略弃权。如果有任何 ACCESS_DENIED 投票，它将拒绝访问。与其他实现一样，如果所有投票者都弃权，则有一个参数控制行为。

可以实现一个自定义 AccessDecisionManager，它以不同的方式记录投票。例如，来自特定 AccessDecisionVoter 的投票可能会获得额外的权重，而来自特定投票者的拒绝投票可能会产生否决效果。

##### RoleVoter
Spring Security 提供的最常用的 AccessDecisionVoter 是简单的 RoleVoter，它将配置属性视为简单的角色名，如果用户被分配了该角色，则通过投票来授予访问权。

如果任何 ConfigAttribute 以前缀 ROLE_ 开头，它就会进行投票。如果有一个 GrantedAuthority 返回一个字符串表示（通过 getAuthority() 方法）恰好等于一个或多个以 ROLE_ 开头的 ConfigAttributes，那么它将投票授予访问权。如果以 ROLE_ 开头的 ConfigAttribute 没有完全匹配，RoleVoter 将投票拒绝访问。如果没有 ConfigAttribute 以 ROLE_ 开头，投票者将弃权。

##### AuthenticatedVoter
我们已经隐式看到的另一个投票者是 AuthenticatedVoter，它可用于区分匿名的、完全身份验证的和 remember-me 身份验证的用户。许多站点在 remember-me 身份验证下允许某些有限的访问，但要求用户通过登录进行完全访问来确认自己的身份。

当我们使用属性 IS_AUTHENTICATED_ANONYMOUSLY 来授予匿名访问时，此属性由 AuthenticatedVoter 处理。

##### 自定义投票者
显然，还可以实现一个自定义的 AccessDecisionVoter，并且可以在其中加入任何想要的访问控制逻辑。它可能特定于的应用程序（与业务逻辑相关），也可能实现一些安全管理逻辑。例如，在 Spring web 站点上有一篇[博客文章](https://spring.io/blog/2009/01/03/spring-security-customization-part-2-adjusting-secured-session-in-real-time)，其中描述了如何使用投票者实时拒绝帐户已挂起的用户访问。

### 后调用处理
虽然 AccessDecisionManager 是由 AbstractSecurityInterceptor 在继续安全对象调用之前调用的，但是一些应用程序需要一种修改安全对象调用实际返回的对象的方法。虽然可以轻松实现自己的 AOP 关注点来实现这一点，但 Spring Security 提供了一个方便的挂钩，它有几个与 ACL 功能集成的具体实现。

After Invocation 实现演示了 Spring Security 的 AfterInvocationManager 及其具体实现。

![after-invocation](/images/springsecurity/after-invocation.png)

像 Spring Security 的许多其他部分一样， AfterInvocationManager 有一个具体实现 AfterInvocationProviderManager，它轮询 AfterInvocationProvider 的列表。允许每个 AfterInvocationProvider 修改返回对象或抛出 AccessDeniedException。实际上，多个提供程序可以修改对象，因为上一个提供程序的结果将传递给列表中的下一个提供程序。

注意，如果使用的是 AfterInvocationManager，仍然需要配置属性来允许 MethodSecurityInterceptor 的 AccessDecisionManager 允许一个操作。如果使用的是典型的包含 Spring Security 的 AccessDecisionManager 实现，没有为特定的安全方法调用定义配置属性将导致每个 AccessDecisionVoter 放弃投票。反过来，如果 AccessDecisionManager 的属性 allowIfAllAbstainDecisions 为 false，则抛出 AccessDeniedException。 可以通过将 allowIfAllAbstainDecisions 设置为 true(尽管通常不建议这样做) 或简单地确保 AccessDecisionVoter 将投票授予访问权限的配置属性至少有一个来避免此潜在问题。后一种（推荐的）方法通常通过 ROLE_USER 或 ROLE_AUTHENTICATED 配置属性实现。

### 层次化角色
应用程序中的特定角色应自动“包含”其他角色，这是一个常见的要求。例如，在具有“admin”和“user”角色概念的应用程序中，我们可能希望管理员能够执行普通用户可以执行的所有操作。为了实现这一点，可以确保所有管理员用户也被分配了“user”角色。或者，可以修改要求“user”角色也包括“admin”角色的每个访问约束。如果应用程序中有很多不同的角色，这可能会变得相当复杂。

使用角色层次结构允许配置哪些角色（或权限）应该包含其他角色。Spring Security 的 RoleVoter 的扩展版本 RoleHierarchyVoter 配置了一个 RoleHierarchy，从这个 RoleHierarchy 中可以获得分配给用户的所有“可访问权限”。典型的配置可能如下所示：

```xml
<project>
    <bean id="roleVoter" class="org.springframework.security.access.vote.RoleHierarchyVoter">
        <constructor-arg ref="roleHierarchy" />
    </bean>
    <bean id="roleHierarchy" class="org.springframework.security.access.hierarchicalroles.RoleHierarchyImpl">
        <property name="hierarchy">
            <value>
                ROLE_ADMIN > ROLE_STAFF
                ROLE_STAFF > ROLE_USER
                ROLE_USER > ROLE_GUEST
            </value>
        </property>
    </bean>
</project>
```

在这里，我们有四个角色在层次结构中，角色是 ROLE_ADMIN、ROLE_STAFF、ROLE_USER、ROLE_GUEST。当根据配置有上述 RoleHierarchyVoter 的 AccessDecisionManager 评估安全约束时，使用 ROLE_ADMIN 进行身份验证的用户将表现为拥有所有四个角色。> 符号可以被认为是“包含”的意思。

角色层次结构提供了一种方便的方法，可以简化应用程序的访问控制配置数据和/或减少需要分配给用户的权限数。对于更复杂的需求，您可能希望定义应用程序所需的特定访问权限与分配给用户的角色之间的逻辑映射，并在加载用户信息时在两者之间进行转换。

## 使用 FilterSecurityInterceptor 授权 HttpServletRequest
本节通过深入研究授权如何在基于 Servlet 的应用程序中工作来构建 Servlet 体系结构和实现。

FilterSecurityInterceptor 为 HttpServletRequests 提供授权。它作为安全过滤器之一插入到 FilterChainProxy 中。

![filtersecurityinterceptor](/images/springsecurity/filtersecurityinterceptor.png)

1. 首先，FilterSecurityInterceptor 从 SecurityContextHolder 获得 Authentication 。

2. 其次，FilterSecurityInterceptor 从传递到 FilterSecurityInterceptor 的 HttpServletRequest、HttpServletResponse 和 FilterChain 创建一个 FilterInvocation。

3. 接下来，它将 FilterInvocation 传递给 SecurityMetadataSource 来获取 ConfigAttributes。

4. 最后，它将 Authentication、FilterInvocation 和 ConfigAttributes 传递给 AccessDecisionManager。

5. 如果拒绝授权，则抛出 AccessDeniedException。在本例中，ExceptionTranslationFilter 处理 AccessDeniedException。

6. 如果允许访问，FilterSecurityInterceptor 将继续使用 FilterChain，它允许应用程序正常处理。

默认情况下，Spring Security 的授权将要求对所有请求进行身份验证。显式配置如下：

```java
rotected void configure(HttpSecurity http) throws Exception {
    http
        // ...
        .authorizeRequests(authorize -> authorize
            .anyRequest().authenticated()
        );
}
```

我们可以通过按照优先顺序添加更多的规则来配置 Spring Security，使其具有不同的规则。

```java
protected void configure(HttpSecurity http) throws Exception {
    http
        // ...
        .authorizeRequests(authorize -> authorize                                  
            .mvcMatchers("/resources/**", "/signup", "/about").permitAll()         
            .mvcMatchers("/admin/**").hasRole("ADMIN")                             
            .mvcMatchers("/db/**").access("hasRole('ADMIN') and hasRole('DBA')")   
            .anyRequest().denyAll()                                                
        );
}
```

1. 指定了多个授权规则。每条规则都是按照其声明的顺序考虑的。

2. 我们指定了多个用户可以访问的 URL 模式。具体来说，如果 URL 以“/resources/”、等于“/signup”或等于“/about”开头，任何用户都可以访问请求。

3. 任何以“/admin/”开头的 URL 将被限制为具有“ROLE_ADMIN”角色的用户。由于我们正在调用 hasRole 方法，所以不需要指定 “ROLE_” 前缀。

4. 任何以“/db/”开头的 URL 都要求用户同时具有“ROLE_ADMIN”和“ROLE_DBA”。由于我们使用的是 hasRole 表达式，所以不需要指定“ROLE_”前缀。

5. 任何尚未匹配的 URL 都将被拒绝访问。如果不想意外地忘记更新授权规则，这是一个很好的策略。

## 基于表达式的访问控制
Spring Security 3.0 引入了使用 Spring EL 表达式作为授权机制的能力，此外还简单使用了前面提到的配置属性和访问决策投票者。基于表达式的访问控制建立在同一体系结构上，但允许将复杂的布尔逻辑封装在单个表达式中。

### 概述
Spring Security 使用 Spring EL 来支持表达式。表达式使用“根对象”作为计算上下文的一部分进行计算。Spring Security 使用 web 的特定类和方法安全性作为根对象，以便提供内置表达式和对当前主体等值的访问。

#### 常见的内置的表达式
表达式根对象的基类是 SecurityExpressionRoot。这提供了一些在 web 和方法安全性中都可用的公共表达式。

Expression | Description
:-: | :-:
hasRole(String role) | 如果当前主体具有指定的角色，则返回 true。<br>例如，hasRole('admin')<br>默认情况下，如果提供的角色不是以 'ROLE_' 开头，则将添加该角色。这可以通过修改 DefaultWebSecurityExpressionHandler 上的 defaultRolePrefix 进行自定义。
hasAnyRole(String…​ roles) | 如果当前主体具有任何提供的角色（以逗号分隔的字符串列表形式给出），则返回 true。<br>例如，hasAnyRole('admin','user')<br>默认情况下，如果提供的角色不是以 'ROLE_' 开头，则将添加该角色。这可以通过修改 DefaultWebSecurityExpressionHandler 上的 defaultRolePrefix 进行自定义。
hasAuthority(String authority) | 如果当前主体具有指定的权限，则返回 true。<br>例如，hasAuthority('read')
hasAnyAuthority(String…​ authorities) | 如果当前主体具有提供的任何权限（以逗号分隔的字符串列表形式给出），则返回 true<br>例如，hasAnyAuthority('read'，'write')
principal | 允许直接访问表示当前用户的主体对象
authentication | 允许直接访问从 SecurityContext 获取的当前身份验证对象
permitAll | 总是等于 true
denyAll | 总是等于 false
isAnonymous() | 如果当前主体是匿名用户，则返回 true
isRememberMe() | 如果当前主体是 remember-me 用户，则返回 true
isAuthenticated() | 如果用户不是匿名的，则返回 true
isFullyAuthenticated() | 如果用户不是匿名用户或 remember-me 用户，则返回 true
hasPermission(Object target, Object permission) | 如果用户有权访问给定权限的所提供目标，则返回 true。例如，hasPermission(domainObject,'read')
hasPermission(Object targetId, String targetType, Object permission) | 如果用户有权访问给定权限的所提供目标，则返回 true。例如， hasPermission(1, 'com.example.domain.Message', 'read')

### Web 安全表达式
要使用表达式来保护单个 URL，首先需要将 <http> 元素中的 use-expressions 属性设置为 true。然后，Spring Security 将期望 <intercept-url> 元素的 access 属性包含 Spring EL 表达式。表达式的值应该是一个布尔值，定义是否允许访问。例如：

```xml
<http>
    <intercept-url pattern="/admin*" access="hasRole('admin') and hasIpAddress('192.168.1.0/24')"/>
    ...
</http>
```

在这里，我们定义了应用程序的“admin”区域(由 URL 模式定义)应该只对具有授予的权限“admin”且其 IP 地址与本地子网匹配的用户可用。我们已经在前一节中看到了内置的 hasRole 表达式。表达式 hasIpAddress 是一个特定于 web 安全的附加内置表达式。它是由 WebSecurityExpressionRoot 类定义的，在计算 web 访问表达式时，该类的实例用作表达式根对象。该对象还直接在名称 request 下公开了 HttpServletRequest 对象，因此可以在表达式中直接调用请求。如果正在使用表达式，则将向命名空间使用的 AccessDecisionManager 中添加 WebExpressionVoter。因此，如果不使用命名空间并希望使用表达式，则必须将其中一个添加到配置中。

#### 在 Web 安全表达式中引用 Beans
如果希望扩展可用的表达式，那么可以很容易地引用公开的任何 Spring Bean。例如，假设有一个名为 webSecurity 的 Bean，它包含以下方法签名：

```java
public class WebSecurity {
    public boolean check(Authentication authentication, HttpServletRequest request) {
        ...
    }
}
```

可以参考以下方法：

```java
http
    .authorizeRequests(authorize -> authorize
        .antMatchers("/user/**").access("@webSecurity.check(authentication,request)")
        ...
    )
```

#### Web 安全表达式中的路径变量
有时能够引用 URL 中的路径变量是很好的。例如，考虑一个 RESTful 应用程序，它从格式为 /user/{userId} 的 URL 路径中查找用户的 id。



通过将路径变量放在模式中，可以很容易地引用它。例如，如果有一个名为 webSecurity 的 Bean，它包含以下方法签名：

```java
public class WebSecurity {
    public boolean checkUserId(Authentication authentication, int id) {
        ...
    }
}
```

可以参考以下方法：

```java
http
    .authorizeRequests(authorize -> authorize
        .antMatchers("/user/{userId}/**").access("@webSecurity.checkUserId(authentication,#userId)")
        ...
    );
```

在这两种配置中，匹配的 URLs 都会将 path 变量（并将其转换）传递到 checkUserId 方法中。例如，如果 URL 是 /user/123/resource，那么传入的 id 将是 123。

### 方法安全表达式
方法安全性比简单的允许或拒绝规则稍微复杂一些。Spring Security 3.0 引入了一些新的注解，以便全面支持表达式的使用。

#### @Pre 和 @Post 注解
有四个注解支持表达式属性，以允许调用前和调用后的授权检查，还支持对提交的集合参数或返回值进行过滤。它们是 @PreAuthorize、@PreFilter、@PostAuthorize 和 @PostFilter。它们的使用是通过 global-method-security 命名空间元素启用的：

```xml
<global-method-security pre-post-annotations="enabled"/>
```

##### 使用 @PreAuthorize 和 @PostAuthorize 进行访问控制
最明显有用的注解是 @PreAuthorize，它决定是否可以实际调用方法。例如（来自“Contacts”示例应用程序）

```java
@PreAuthorize("hasRole('USER')")
public void create(Contact contact);
```

这意味着只允许具有“ROLE_USER”角色的用户访问。显然，使用传统配置和所需角色的简单配置属性也可以很容易地实现相同的功能。但：

```java
@PreAuthorize("hasPermission(#contact, 'admin')")
public void deletePermission(Contact contact, Sid recipient, Permission permission);
```

在这里，我们实际使用一个方法参数作为表达式的一部分来决定当前用户是否具有给定联系人的“admin”权限。内置的 hasPermission() 表达式通过应用程序上下文链接到  Spring Security ACL 模块，如下所示。可以通过名称作为表达式变量访问任何方法参数。

Spring Security 可以通过多种方式解析方法参数。Spring Security 使用 DefaultSecurityParameterNameDiscoverer 来发现参数名称。默认情况下，将对整个方法尝试以下选项。

* 如果 Spring Security 的 @P 注解出现在方法的单个参数上，那么将使用该值。这对于在 JDK 8 之前使用 JDK 编译的接口很有用，JDK 8 不包含任何有关参数名的信息。例如：

```java
import org.springframework.security.access.method.P;

...

@PreAuthorize("#c.name == authentication.name")
public void doSomething(@P("c") Contact contact);
```

在幕后，这个使用是使用 AnnotationParameterNameDiscoverer 实现的，可以对其进行自定义以支持任何指定注解的 value 属性。

* 如果 Spring Data 的 @Param 注解出现在该方法的至少一个参数上，则将使用该值。这对于在 JDK 8 之前使用 JDK 编译的接口很有用，JDK 8 不包含任何有关参数名的信息。例如：

```java
import org.springframework.data.repository.query.Param;

...

@PreAuthorize("#n == authentication.name")
Contact findContactByName(@Param("n") String name);
```

在幕后，这种用法是使用 AnnotationParameterNameDiscoverer 实现的，可以对其进行自定义以支持任何指定注解的 value 属性。

* 如果使用 JDK 8 编译带有 -parameters 参数的源代码，并且使用 Spring4+，则使用标准 JD K反射 API 来发现参数名。这对类和接口都有效。 

* 最后，如果代码是用调试符号编译的，那么将使用调试符号发现参数名。这对接口不起作用，因为它们没有关于参数名的调试信息。对于接口，必须使用注释或 JDK 8方法。 

任何 Spring-EL 功能在表达式中都是可用的，因此还可以访问参数的属性。例如，如果希望某个特定方法只允许访问其用户名与联系人的用户名匹配的用户，可以编写

```java
@PreAuthorize("#contact.name == authentication.name")
public void doSomething(Contact contact);
```

在这里，我们访问另一个内置表达式 authentication，它是存储在安全上下文中的 Authentication。还可以使用表达式 principal 直接访问其“principal”属性。该值通常是一个 UserDetails 实例，因此可以使用类似 principal.username 或 principal.enabled 这样的表达式。

不太常见的是，可能希望在调用方法之后执行访问控制检查。这可以使用 @PostAuthorize 注解来实现。要访问方法的返回值，在表达式中使用内置的名称 returnObject。

##### 使用 @PreFilter 和 @PostFilter 过滤
我们可能已经知道，Spring Security 支持对集合和数组进行过滤，现在可以使用表达式来实现这一点。这通常是对方法的返回值执行的。例如：

```java
@PreAuthorize("hasRole('USER')")
@PostFilter("hasPermission(filterObject, 'read') or hasPermission(filterObject, 'admin')")
public List<Contact> getAll();
```

当使用 @PostFilter 注解时，Spring Security 会遍历返回的集合，并删除所提供表达式为 false 的任何元素。名称 filterObject 引用集合中的当前对象。也可以在方法调用之前使用 @PreFilter 进行过滤，尽管这是一个不太常见的要求。语法是一样的，但是如果有多个集合类型的参数，则必须使用该注解的 filterTarget 属性按名称选择一个。

注意，过滤显然不能替代优化数据检索查询。如果过滤大型集合并删除许多项，那么这可能是低效的。

#### 内置表达式
有一些特定于方法安全性的内置表达式，我们已经在上面的使用中看到了。filterTarget 和returnValue 值非常简单，但是 hasPermission() 表达式的使用值得仔细研究。

##### PermissionEvaluator 
hasPermission() 表达式被委托给 PermissionEvaluator 的实例。它旨在在表达式系统和 Spring Security 的 ACL 系统之间建立桥梁，允许基于抽象权限在域对象上指定授权约束。它对 ACL 模块没有显式的依赖关系，因此如果需要，可以将其替换为另一个实现。该接口有两个方法：

```java
boolean hasPermission(Authentication authentication, Object targetDomainObject, Object permission);

boolean hasPermission(Authentication authentication, Serializable targetId, String targetType, Object permission);
```

它直接映射到表达式的可用版本，但未提供第一个参数（Authentication 对象）除外。第一种方法用于已加载访问控制的域对象的情况。如果当前用户具有该对象的给定权限，那么表达式将返回 true。第二个版本用于没有加载对象，但知道其标识符的情况。还需要域对象的抽象“type”说明符，以便加载正确的 ACL 权限。这通常是对象的 Java 类，但只要与加载权限的方式一致，就不必是这样。

要使用 hasPermission() 表达式，必须在应用程序上下文中显式配置 PermissionEvaluator。看起来像这样：

```xml
<project>
    <security:global-method-security pre-post-annotations="enabled">
    <security:expression-handler ref="expressionHandler"/>
    </security:global-method-security>
    
    <bean id="expressionHandler" class="org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler">
        <property name="permissionEvaluator" ref="myPermissionEvaluator"/>
    </bean>
</project>
```

其中 myPermissionEvaluator 是实现 PermissionEvaluator 的 bean。通常这将是来自 ACL 模块的实现，该模块称为 AclPermissionEvaluator。

##### 方法安全元注解
可以使用元注解实现方法安全性，从而使代码更具可读性。如果发现在整个代码库中重复使用相同的复杂表达式，那么这将特别方便。例如，考虑以下几点：

```java
@PreAuthorize("#contact.name == authentication.name")
```

我们可以创建一个可以替代的元注解，而不是在任何地方重复此操作。

```java
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("#contact.name == authentication.name")
public @interface ContactPermission {}
```

元注解可以用于任何 Spring Security 方法的安全注解。为了保持与规范的兼容性，JSR-250 注解不支持元注解。

## 安全对象的实现
### AOP Alliance (MethodInvocation) 安全拦截器
在 Spring Security 2.0 之前，保护 MethodInvocation 需要大量的公式化配置。现在推荐的方法安全方式是使用命名空间配置。通过这种方式，方法安全基础设施 beans 将自动配置，因此实际上不需要了解实现类。我们将简要介绍这里涉及的类。

方法安全是使用一个 MethodSecurityInterceptor 来实现的，它保护 MethodInvocation。根据配置方法，一个拦截器可能是特定于一个 bean 的，也可能是在多个 beans 之间共享的。拦截器使用一个 MethodSecurityMetadataSource 实例来获取应用于特定方法调用的配置属性。MapBasedMethodSecurityMetadataSource 用于存储由方法名（可以是通配符）键入的配置属性，当使用 <intercept-methods> 或 <protect-point> 元素在应用程序上下文中定义属性时，将在内部使用这些属性。其他实现将用于处理基于注解的配置。

#### 显式 MethodSecurityInterceptor 配置
当然，可以在的应用程序上下文中直接配置一个 MethodSecurityIterceptor 来使用 Spring AOP 的代理机制：

```xml
<project>
    <bean id="bankManagerSecurity" class="org.springframework.security.access.intercept.aopalliance.MethodSecurityInterceptor">
    <property name="authenticationManager" ref="authenticationManager"/>
    <property name="accessDecisionManager" ref="accessDecisionManager"/>
    <property name="afterInvocationManager" ref="afterInvocationManager"/>
    <property name="securityMetadataSource">
        <sec:method-security-metadata-source>
        <sec:protect method="com.mycompany.BankManager.delete*" access="ROLE_SUPERVISOR"/>
        <sec:protect method="com.mycompany.BankManager.getBalance" access="ROLE_TELLER,ROLE_SUPERVISOR"/>
        </sec:method-security-metadata-source>
    </property>
    </bean>
</project>
```

### AspectJ (JoinPoint) 安全拦截器
AspectJ 安全拦截器与上一节讨论的 AOP Alliance 安全拦截器非常相似。实际上，我们只会在这一节中讨论差异。

AspectJ 拦截器被命名为 AspectJSecurityInterceptor。与 AOP Alliance 安全拦截器不同，AOP Alliance 安全拦截器依赖于 Spring 应用程序上下文通过代理编织在安全拦截器中，而 AspectJSecurityInterceptor 则是通过 AspectJ 编译器编织在安全拦截器中。在同一个应用程序中同时使用这两种类型的安全拦截器并不少见，AspectJSecurityInterceptor 用于域对象实例安全，而 AOP Alliance MethodSecurityInterceptor 用于服务层安全。

我们首先考虑如何在 Spring 应用程序上下文中配置 AspectJSecurityInterceptor：

```xml
<project>
    <bean id="bankManagerSecurity" class="org.springframework.security.access.intercept.aspectj.AspectJMethodSecurityInterceptor">
    <property name="authenticationManager" ref="authenticationManager"/>
    <property name="accessDecisionManager" ref="accessDecisionManager"/>
    <property name="afterInvocationManager" ref="afterInvocationManager"/>
    <property name="securityMetadataSource">
        <sec:method-security-metadata-source>
        <sec:protect method="com.mycompany.BankManager.delete*" access="ROLE_SUPERVISOR"/>
        <sec:protect method="com.mycompany.BankManager.getBalance" access="ROLE_TELLER,ROLE_SUPERVISOR"/>
        </sec:method-security-metadata-source>
    </property>
    </bean>
</project>
```

可以看到，除了类名之外，AspectJSecurityInterceptor 与 AOP Alliance 安全拦截器完全相同。实际上，这两个拦截器可以共享同一个 securityMetadataSource，因为 SecurityMetadataSource 使用 java.lang.reflect 方法而不是特定于 AOP 库的类。当然，访问决策可以访问相关的AOP库特定的调用（比如 MethodInvocation或JoinPoint），因此在做出访问决策时可以考虑一系列附加条件（例如方法参数）。

接下来需要定义 AspectJ 方面。例如：

```java
package org.springframework.security.samples.aspectj;

import org.springframework.security.access.intercept.aspectj.AspectJSecurityInterceptor;
import org.springframework.security.access.intercept.aspectj.AspectJCallback;
import org.springframework.beans.factory.InitializingBean;

public aspect DomainObjectInstanceSecurityAspect implements InitializingBean {

    private AspectJSecurityInterceptor securityInterceptor;

    pointcut domainObjectInstanceExecution(): target(PersistableEntity)
        && execution(public * *(..)) && !within(DomainObjectInstanceSecurityAspect);

    Object around(): domainObjectInstanceExecution() {
        if (this.securityInterceptor == null) {
            return proceed();
        }

        AspectJCallback callback = new AspectJCallback() {
            public Object proceedWithObject() {
                return proceed();
            }
        };

        return this.securityInterceptor.invoke(thisJoinPoint, callback);
    }

    public AspectJSecurityInterceptor getSecurityInterceptor() {
        return securityInterceptor;
    }

    public void setSecurityInterceptor(AspectJSecurityInterceptor securityInterceptor) {
        this.securityInterceptor = securityInterceptor;
    }

    public void afterPropertiesSet() throws Exception {
        if (this.securityInterceptor == null)
            throw new IllegalArgumentException("securityInterceptor required");
        }
    }
}
```

在上面的示例中，安全拦截器将应用于 PersistableEntity 的每个实例，PersistableEntity 是一个未显示的抽象类（可以使用任何其他喜欢的类或切入点表达式）。对于那些好奇的人来说，AspectJCallback 是必需的，因为 proceed(); 语句只有在 around() 主体中才有特殊意义。AspectJSecurityInterceptor 在希望目标对象继续时调用这个匿名的 AspectJCallback 类。

需要配置 Spring 来加载切面并将它与 AspectJSecurityInterceptor 连接起来。实现这一点的 bean 声明如下所示：

```xml
<bean id="domainObjectInstanceSecurityAspect"
    class="security.samples.aspectj.DomainObjectInstanceSecurityAspect"
    factory-method="aspectOf">
<property name="securityInterceptor" ref="bankManagerSecurity"/>
</bean>
```

就这样！现在，可以在应用程序中的任何地方创建 beans，使用我们认为合适的任何方法（例如 new Person();），它们将应用安全拦截器。

## 方法安全
从 2.0 版本开始，Spring Security 大大改进了对向服务层方法添加安全性的支持。它提供了对 JSR-250 注解安全性以及框架的原始 @Secured 注解的支持。从 3.0 开始，我们还可以使用新的基于表达式的注解。我们可以将安全性应用到单个 bean，使用 intercept-methods 元素来修饰 bean 声明，或者可以使用 AspectJ 风格的切入点来保护整个服务层的多个 beans。

### EnableGlobalMethodSecurity
我们可以在任何 @Configuration 实例上使用 @EnableGlobalMethodSecurity 注解来启用基于注解的安全。例如，下面的代码将启用 Spring Security 的 @Secure 注解。

```java
@EnableGlobalMethodSecurity(securedEnabled = true)
public class MethodSecurityConfig {
// ...
}
```

向方法（在类或接口上）添加注解将相应地限制对该方法的访问。Spring Security 的原生注解支持为该方法定义了一组属性。这些将被传递给 AccessDecisionManager，让它做出实际的决定：

```java
public interface BankService {

@Secured("IS_AUTHENTICATED_ANONYMOUSLY")
public Account readAccount(Long id);

@Secured("IS_AUTHENTICATED_ANONYMOUSLY")
public Account[] findAccounts();

@Secured("ROLE_TELLER")
public Account post(Account account, double amount);
}
```

可以使用以下命令启用对 JSR-250 注解的支持：

```java
@EnableGlobalMethodSecurity(jsr250Enabled = true)
public class MethodSecurityConfig {
// ...
}
```

这些都是基于标准的，允许应用简单的基于角色的约束，但是没有 Spring Security 的原生注解的能力。要使用新的基于表达式的语法，我们将使用：

```java
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class MethodSecurityConfig {
// ...
}
```

等价的 Java 代码是：

```java
public interface BankService {

@PreAuthorize("isAnonymous()")
public Account readAccount(Long id);

@PreAuthorize("isAnonymous()")
public Account[] findAccounts();

@PreAuthorize("hasAuthority('ROLE_TELLER')")
public Account post(Account account, double amount);
}
```

### GlobalMethodSecurityConfiguration
有时，我们可能需要执行比使用 @EnableGlobalMethodSecurity 注解所允许的更复杂的操作。对于这些实例，可以扩展 GlobalMethodSecurityConfiguration，以确保 @EnableGlobalMethodSecurity 注解出现在子类中。例如，如果我们想提供一个自定义的 MethodSecurityExpressionHandler，可以使用以下配置：

```java
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class MethodSecurityConfig extends GlobalMethodSecurityConfiguration {
    @Override
    protected MethodSecurityExpressionHandler createExpressionHandler() {
        // ... create and return custom MethodSecurityExpressionHandler ...
        return expressionHandler;
    }
}
```

### 使用 protect-pointcut 添加安全切入点
protect-pointcut 的用途特别强大，因为它允许我们仅通过一个简单的声明就可以将安全性应用到许多 beans。考虑以下示例：

```xml
<global-method-security>
    <protect-pointcut expression="execution(* com.mycompany.*Service.*(..))" access="ROLE_USER"/>
</global-method-security>
```

这将保护在应用程序上下文中声明的 beans 上的所有方法，这些 beans 的类位于 com.mycompany 包中，其类名以 “Service” 结尾。只有具有 ROLE_USER 角色的用户才能调用这些方法。与 URL 匹配一样，在切入点列表中，最具体的匹配必须排在第一位，因为将使用第一个匹配表达式。安全注解优先于切入点。

## 参考资料
1. [Authorization](https://docs.spring.io/spring-security/site/docs/5.3.0.RELEASE/reference/html5/#servlet-authorization)