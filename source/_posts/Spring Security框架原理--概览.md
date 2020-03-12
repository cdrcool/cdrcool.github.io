---
title: Spring Security框架原理--概览
date: 2019-05-23 15:33:57
categories: Spring Security
tags:
    - Spring Security
---
# SecurityFilterChain
![Spring Security 过滤器链 UML](/images/springsecurity/Spring%20Security过滤器链UML.png)

* 表单认证
![Spring Security过滤器（表单认证）](/images/springsecurity/Spring%20Security过滤器链-表单认证.png)

* Http Basic认证
![Spring Security过滤器（Http Basic认证）](/images/springsecurity/Spring%20Security过滤器链-Http%20Basic认证.png)

# AuthenticationManager
![AuthenticationManager UML](/images/springsecurity/AuthenticationManager%20UML.png)

# Authentication
![Authentication UML](/images/springsecurity/Authentication%20UML.png)

* UsernamePasswordAuthenticationToken
    + JaasAuthenticationToken
* AnonymousAuthenticationToken
* RememberMeAuthenticationToken
* PreAuthenticatedAuthenticationToken
* BearerTokenAuthenticationToken
* OAuth2AuthenticationToken
* OAuth2LoginAuthenticationToken
* OAuth2AuthorizationCodeAuthenticationToken
* OpenIDAuthenticationToken
* CasAuthenticationToken
* CasAssertionAuthenticationToken
* RunAsUserToken
* TestingAuthenticationToken

* AbstractOAuth2TokenAuthenticationToken
    + JwtAuthenticationToken
    + OAuth2IntrospectionAuthenticationToken

# AuthenticationProvider
![AuthenticationProvider UML](/images/springsecurity/AuthenticationProvider%20UML.png)

**UsernamePasswordAuthenticationToken:** 
* AbstractUserDetailsAuthenticationProvider
 + DaoAuthenticationProvider
* AbstractJaasAuthenticationProvider
 + JaasAuthenticationProvider
 + DefaultJaasAuthenticationProvider
* RemoteAuthenticationProvider
* AbstractLdapAuthenticationProvider
 + ActiveDirectoryLdapAuthenticationProvider
 + LdapAuthenticationProvider

* AnonymousAuthenticationProvider
* RememberMeAuthenticationProvider

**PreAuthenticatedAuthenticationToken:** 
* PreAuthenticatedAuthenticationProvider
* GoogleAccountsAuthenticationProvider

**BearerTokenAuthenticationToken:** 
* JwtAuthenticationProvider
* OAuth2IntrospectionAuthenticationProvider

**OAuth2LoginAuthenticationToken:** 
* OAuth2LoginAuthenticationProvider
* OidcAuthorizationCodeAuthenticationProvider

* OAuth2AuthorizationCodeAuthenticationProvider
* OpenIDAuthenticationProvider
* CasAuthenticationProvider
* RunAsImplAuthenticationProvider
* TestingAuthenticationProvider

# SecurityContext
![SecurityContext UML](/images/springsecurity/SecurityContext%20UML.png)

# UserDetails
![UserDetails](/images/springsecurity/UserDetails.png)

# UserDetailsService
![UserDetailsService UML](/images/springsecurity/UserDetailsService%20UML.png)

# AuthenticationEntryPoint
![AuthenticationEntryPoint](/images/springsecurity/AuthenticationEntryPoint.png)

![AuthenticationEntryPoint UML](/images/springsecurity/AuthenticationEntryPoint%20UML.png)

form login -> LoginUrlAuthenticationEntryPoint
basic login -> BasicAuthenticationEntryPoint
undefined -> Http403ForbiddenEntryPoint
resource server -> OAuth2AuthenticationEntryPoint
