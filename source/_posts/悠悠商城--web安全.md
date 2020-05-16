---
title: 悠悠商城--web安全
date: 2019-07-19 20:39:49
categories: 悠悠商城
tags:
	- Spring Security
---
## 整体流程
![web安全整体流程](/images/yoyo mall/web安全整体流程.png)
实际流程中，Access Token就是JWT token。

## JWT签名
使用RSA非对称加密算法（私钥加密，公钥解密）做jwt签名和校验。
### 创建KeyPair
1. 编程方式
    ```java
    KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
    // 模数bit位数，1024位足够安全
    generator.initialize(1024);
    KeyPair pair = generator.generateKeyPair();
    RSAPrivateKey privateKey = (RSAPrivateKey) pair.getPrivate();
    RSAPublicKey publicKey = (RSAPublicKey) pair.getPublic();
    
    RSAPublicKeySpec publicSpec = new RSAPublicKeySpec(new BigInteger(publicKey.getModulus()), new BigInteger(publicKey.getPublicExponent()));
    RSAPrivateKeySpec privateSpec = new RSAPrivateKeySpec(new BigInteger(privateKey.getModulus()), new BigInteger(privateKey.getPrivateExponent()));
    KeyFactory factory = KeyFactory.getInstance("RSA");
    KeyPair keyPair = new KeyPair(factory.generatePublic(publicSpec), factory.generatePrivate(privateSpec));
    ```
2. 使用java keytool
    ```
    # 创建证书
    keytool -genkey -alias yoyo-mall-certificate -keyalg rsa -keystore E:\keys.keystore -keysize 1024 -validity 36500
    
    # 查看证书库
    keytool -list -keystore E:\keys.keystore
    
    # 导出公钥
    keytool -export -alias yoyo-mall-certificate -keystore E:\keys.keystore -file E:\publickey.cer
    ```
    ![创建证书示例](/images/yoyo mall/创建证书示例.png)
    ```
    ClassPathResource resource = new ClassPathResource(keystoreLocation);
    KeyStoreKeyFactory keyStoreKeyFactory = new KeyStoreKeyFactory(resource, keystorePassword.toCharArray());
    KeyPair keyPair = keyStoreKeyFactory.getKeyPair(certificateAlias, certificatePassword.toCharArray());
    ```

## XSS攻击
XSS攻击，即跨站脚本攻击XSS(Cross Site Scripting)，指恶意攻击者往Web页面里插入恶意Script代码，当用户浏览该页面时，嵌入Web里面的Script代码会被执行，从而达到恶意攻击用户的目的。
XSS的攻击目标是为了盗取客户端的cookie或者其他网站用于识别客户端身份的敏感信息。获取到合法用户的信息后，攻击者甚至可以假冒最终用户与网站进行交互。
![XSS攻击示例流程](/images/yoyo mall/XSS攻击示例流程.png)

### 防护措施
1. response添加header:`X-Xss-Protection:1; mode=block`，该措施只能防护反射行XSS，且Spring Cloud Gateway已默认添加。

## CSRF防护
CSRF，即跨站请求伪造（Cross Site Request Forgery）。示例流程如下：
![csrf示例流程](/images/yoyo mall/csrf示例流程.png)

由于CSRF基于cookie-session，而jwt token是无状态的，所以jwt token天然支持CSRF防护。

## Session固定攻击
Session固定攻击（Session Fixation Attack），是利用应用系统在服务器的会话ID固定不变机制，借助他人用相同的会话ID获取认证和授权，然后利用该会话ID劫持他人的会话以成功冒充他人，造成会话固定攻击。示例流程如下：
1. 攻击者访问网站 http:///www.bank.com ，获取他自己的session id，如：SID=123；
2. 攻击者给目标用户发送链接，并带上自己的session id，如：http:///www.bank.com/?SID=123 ；
3. 目标用户点击了 http:///www.bank.com/?SID=123 ，像往常一样，输入自己的用户名、密码登录到网站；
4. 由于服务器的session id不改变，现在攻击者点击 http:///www.bank.com/?SID=123 ，他就拥有了目标用户的身份，可以为所欲为了。

由于为用到session，所以也不需做额外防护。

## 集成网关（Spring Cloud Gateway）
```java
@EnableWebFluxSecurity
public class SecurityConfiguration {
    /**
     * swagger文档相关请求地址无需身份验证
     */
    private static final String[] SWAGGER_URLS = new String[]{
            "/swagger-ui.html",
            "/webjars/springfox-swagger-ui/**",
            "/*/v2/api-docs/**",
            "/swagger-resources/**"
    };

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        http
                .authorizeExchange()
                .pathMatchers(SWAGGER_URLS).permitAll()
                .anyExchange().authenticated()
                .and()
                .oauth2ResourceServer().jwt();
        return http.build();
    }
}
```

## 集成Feign
```java
@Component
public class OAuth2FeignRequestInterceptor implements RequestInterceptor {
    private static final String BEARER = "Bearer";

    @Override
    public void apply(RequestTemplate requestTemplate) {
        requestTemplate.header(HttpHeaders.AUTHORIZATION, extract());
    }

    private String extract() {
        String token = ((Jwt) SecurityContextHolder.getContext().getAuthentication().getPrincipal()).getTokenValue();
        return String.format("%s %s", BEARER, token);
    }
}
```
```
hystrix.command.default.execution.isolation.strategy: SEMAPHORE
```