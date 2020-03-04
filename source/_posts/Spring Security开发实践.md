---
title: Spring Security 开发实践
date: 2020-03-04 23:20:00
categories: Spring Security
tags:
    - Spring Security
---
## Maven 依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

## 基本安全验证
因为 spring-boot-starter-security 是开箱即用的，所以当项目中添加了该依赖之后，便具备了基本安全验证能力。示例：

![基本安全验证](/images/springsecurity/基本安全验证.png)

如上，未登录访问项目时，会自动跳转到一个简易的登录页面，用户名默认是 user，密码会在控制台输出。

## 配置用户
### 内存配置
```java
@Bean
public UserDetailsService userDetailsService() {
    // ensure the passwords are encoded properly
    User.UserBuilder users = User.withDefaultPasswordEncoder();
    InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
    manager.createUser(users.username("user").password("password").roles("USER").build());
    manager.createUser(users.username("admin").password("password").roles("USER","ADMIN").build());
    return manager;
}
```

### 数据库配置
#### HSQLDB
**Maven 依赖：**
```xml
<dependency>
    <groupId>org.hsqldb</groupId>
    <artifactId>hsqldb</artifactId>
    <scope>runtime</scope>
</dependency>
```

**Java 代码：**
```java
@Autowired
public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
    // ensure the passwords are encoded properly
    User.UserBuilder users = User.withDefaultPasswordEncoder();
    auth
            .jdbcAuthentication()
            .dataSource(dataSource)
            // 使用 "org/springframework/security/core/userdetails/jdbc/users.ddl" 自动建表
            .withDefaultSchema()
            // 往 users 表中插入数据
            .withUser(users.username("user").password("password").roles("USER"))
            .withUser(users.username("admin").password("password").roles("USER","ADMIN"));
}
```

#### MySQL
**Maven 依赖：**
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

**配置属性：**
```yaml
spring:
  datasource:
    # always：始终执行初始化；embedded：内存数据库时才执行初始化；never：不执行初始化
    initialization-mode: always
    # 表初始化，默认值 classpath:schema.sql
    schema: classpath:schema.sql
    # 数据初始化，默认值 classpath:data.sql
    data: classpath:data.sql
    # 默认加载 data.sql 或 data-${platform}.sql，默认值 all
    platform: all
    driverClassName: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/demo?useUnicode=true&characterEncoding=utf-8&autoReconnect=true&failOverReadOnly=false&useSSL=true&serverTimezone=Asia/Shanghai
    username: root
    password: admin123
```

**注意：**
* schema.sql 中不能带有注释。
* 如果表名/字段名为 MySQL 关键字，则需要添加 `` 符号。
* 用户登录密码指定加密算法，如 bcrypt、PBKDF2、scrypt、Argon2 等。指定方式为在密码前添加 {${algorithm}} 前缀，${algorithm} 为加密算法的名称。如果确实要明文存储密码，也可以添加前缀 {noop}。

**Java 配置：**
```java
@Bean
public UserDetailsService userDetailsService() {
    return new JdbcUserDetailsManager(dataSource);
}
```

### 密码加密
```java
@Bean
public PasswordEncoder passwordEncoder() {
    // 如果不考虑兼容 Spring Security 5.0 之前的版本，可以直接返回 new BCryptPasswordEncoder()
    DelegatingPasswordEncoder passwordEncoder = (DelegatingPasswordEncoder) PasswordEncoderFactories
            .createDelegatingPasswordEncoder();
    passwordEncoder.setDefaultPasswordEncoderForMatches(new BCryptPasswordEncoder());
    return passwordEncoder;
}
```
**注意：** 在 passwordEncoder() 方法中，也可以直接 `return new BCryptPasswordEncoder()`，之所以这里要**使用 DelegatingPasswordEncoder 并将 BCryptPasswordEncoder 作为默认值给其赋值**，是为了兼容 Spring Security 5.0 之前的版本。（在 Spring Security 5.0 之前，密码是明文存储的，而现在要求密码前面都要带加密算法前缀。）

## 会话管理
### 检测超时
```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/invalidSession").permitAll()
                .anyRequest().authenticated().and()
                .formLogin().and()
                // session 超时后，跳转到 /invalidSession
                .sessionManagement().invalidSessionUrl("/invalidSession")
                .and()
                // 由于配置了 session 超时检测，因此需要保证用户退出时清理掉 JSESSIONID cookie，
                // 不然在用户注销后重新登录时可能会错误地报告错误（测试并未出现该问题）
                .logout().deleteCookies("JSESSIONID");
    }
}
```
session 默认超时时间为 30m，可以通过配置 server.servlet.session.timeout 属性来修改超时时间。

**注意：** 这不能保证与每个 servlet 容器一起工作，所以需要在环境中测试它。

### 并发会话控制
```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/invalidSession").permitAll()
                .anyRequest().authenticated().and()
                .formLogin().and()
                .sessionManagement()
                // 一个用户只能同时在一处登陆。如果存在用户在多处重复登录，前面登录的用户再访问系统时会被重定向到 /invalidSession
                .maximumSessions(1).expiredUrl("/invalidSession");
    }
}
```

当用户在另一处重复登录后，之前登录的用户再访问系统时，默认会收到 “This session has been expired (possibly due to multiple concurrent logins being attempted as the same user).” 错误提示。在上例中，我们是通过设置 expiredSessionStrategy 来将用户重定向到 /invalidSession。

我们也可以通过设置 sessionManagement().maxSessionsPreventsLogin(true) 来阻止用户重复登录。

### 会话固定攻击保护
会话固定攻击是一种潜在的风险，恶意攻击者可能通过访问站点来创建会话，然后说服另一个用户使用相同的会话进行登录(例如，通过将包含会话标识符的链接作为参数发送给他们)。
Spring Security 通过创建一个新会话或在用户登录时更改会话 ID 来自动防止这种情况。

默认情况下，Spring Security 启用了此保护，我们可以通过设置 sessionManagement().sessionFixation().xx 来禁用此保护或调整相关行为。
* sessionFixation().none() 什么都不要做，将保留原来的会议。
* sessionFixation().newSession() 创建一个新的“干净”的会话，而不复制现有的会话数据（Spring 安全相关的属性仍然会被复制）。
* sessionFixation().migrateSession()  创建一个新会话，并将所有现有的会话属性复制到新会话中。这是 Servlet 3.0 或更旧的容器中的默认值。
* sessionFixation().changeSessionId() 不创建新会话。相反，使用 Servlet 容器提供的会话固定保护（HttpServletRequest#changeSessionId()）。此选项仅在 Servlet 3.1 (Java EE 7) 和更新的容器中可用。在旧的容器中指定它将导致异常。这是 Servlet 3.1 和更新的容器中的默认值。

## Remember-me 身份验证
Remember-me 或 persistent-login 身份验证是指 web 站点能够记住不同会话间的主体身份。这通常是通过向浏览器发送 cookie 来完成的，在以后的会话中检测到 cookie 并进行自动登录。

Spring Security 提供了两种具体的 remember-me 实现：
* 使用哈希来保持基于 cookie 的令牌的安全性
* 使用数据库或其他持久存储机制来存储生成的令牌（同样需要使用 cookie）

**注意：** 以上两种实现都需要上下文中存在 UserDetailsService bean。

### 基于哈希的令牌
```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated().and()
                .formLogin().and()
                // 使用哈希来保持基于 cookie 的令牌的安全性
                .rememberMe().key("secret");
    }
}

```

cookie 示例：
![cookie 示例](/images/springsecurity/记住我-cookie示例.png)

这种方式有个弊端，浏览器端要携带的这个 cookie 值在服务端是存放在内存中的，并没有进行持久化，所以如果服务重启后服务器端存储的这个值就会丢失，浏览器端的 remember-me 就会失效。为了解决这个问题就需要将服务器端生成的这个 cookie 值持久化到数据库中。

**注意：** Remember-me 令牌仅在指定的时间段内有效，前提是用户名、密码和密钥不变。值得注意的是，这有一个潜在的安全问题，即捕获的“记住我”令牌在令牌到期之前可以从任何用户代理使用。这与 digest 身份验证的问题相同。如果 principal 知道捕获了令牌，则可以轻松地更改其密码并立即使所有有关问题的“记住我”令牌无效。如果需要更重要的安全性，则应该使用下一节中描述的方法。或者，记住“我”服务根本不应该被使用。

### 持久化令牌
#### 内存存储
```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated().and()
                .formLogin().and()
                // 使用内存数据库来存储生成的令牌
                .rememberMe().tokenRepository(new InMemoryTokenRepositoryImpl());
    }
}
```

#### 数据库存储
```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    private final DataSource dataSource;

    public SecurityConfiguration(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated().and()
                .formLogin().and()
                // 使用数据库来存储生成的令牌
                .rememberMe().tokenRepository(tokenRepository());
    }

    @Bean
    public PersistentTokenRepository tokenRepository() {
        JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
        tokenRepository.setDataSource(dataSource);
        // 自动创建表，仅在首次启动时开启
        tokenRepository.setCreateTableOnStartup(true);
        return tokenRepository;
    }
}
```

在上例中，我们采用了首次启动时自动建表的方式，也可以手动建表，sql 脚本如下：

```sql
create table persistent_logins
(
    series    varchar(64) not null comment '主键',
    username  varchar(64) not null comment '用户名',
    token     varchar(64) not null comment '访问令牌',
    last_used timestamp   not null comment '最后使用时间',
    PRIMARY KEY (`series`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;
```

持久化 token 示例：
![持久化 token 示例](/images/springsecurity/记住我-持久化token示例.png)

## 参考资料
1. [Spring-Security的Password Encoding](https://blog.csdn.net/song_java/article/details/86309907)
2. [spring security 4.0 教程 步步深入 6](https://blog.csdn.net/chemmuxin1993/article/details/53019919)