---
title: Spring Cloud OpenFeign 开发指南
date: 2020-02-28 18:42:00
categories: Spring Cloud
---
Feign 是一个声明式 web 服务客户端，使用它能让编写 web 服务客户端更加简单。Feign 的使用方法是定义一个接口，然后在上面添加注解。它支持可插入的注解支持，包括 Feign 注解和 JAX-RS 注解，同时也支持可插拔的编码器和解码器。Spring Cloud 增加了对 Spring MVC 注解的支持，并支持使用 Spring Web 中默认使用的 HttpMessageConverters。Spring Cloud 集成了 Ribbon 和 Eureka，以及 Spring Cloud LoadBalancer，从而可以使用 Feign 来实现 HTTP 客户端负载均衡。

## 引入 Feign
要在项目中引入 Feign，需要添加以下依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

**使用示例：**
```java
@SpringBootApplication
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}

@FeignClient("stores")
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();

    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);
}
```

在 @FeignClient 注解中，字符串值(上面的 “stores”)是一个任意的客户端名称，用于创建 Ribbon 负载均衡器。还可以使用 url 属性(绝对值或主机名)指定 URL。应用程序上下文中 bean 的名称是接口的完全限定名。要指定自己的别名值，可以使用 @FeignClient 注解的 qualifier 的值。

上面的负载均衡客户端期待发现 “stores” 服务的物理地址。如果我们的应用程序是一个 Eureka 客户端，那么它将在 Eureka 服务注册表中解析服务。如果我们的不想使用Eureka，可以简单地在外部配置中配置一个服务器列表（eg. stores.ribbon.listOfServers: example.com,google.com）。

**注意：** 为了保持向后兼容性，Spring Cloud Netflix Ribbon 被用作默认的负载均衡实现。但是，Spring Cloud Netflix Ribbon 现在处于维护模式，所以建议大家使用 Spring Cloud LoadBalancer 来代替。为此，设置 spring.cloud.loadbalancer.ribbon.enabled=false。

## 覆盖 Feign 默认配置
Spring Cloud Feign 中一个核心的概念就是客户端要有一个名字。每一个客户端随时可以向远程服务发起请求，并且每个服务都可以像使用 @FeignClient 注解一样指定一个名字。Spring Cloud 会将所有的 @FeignClient 组合在一起创建一个新的 ApplicationContext, 并使 用FeignClientsConfiguration对Clients 进行配置。配置中包括一个 feign.Decoder，一个 feign.Encoder 和一个 feign.Contract。

Spring Cloud 允许我们通过 configuration 属性完全控制 Feign 的配置信息，这些配置比 FeignClientsConfiguration 优先级要高：

```java
@FeignClient(name = "stores", configuration = FooConfiguration.class)
public interface StoreClient {
    //..
}
```

在这个例子中，FooConfiguration 中的配置信息会覆盖掉 FeignClientsConfiguration 中对应的配置。

**注意：** FooConfiguration 虽然是个配置类，但是它不应该被主上下文（ApplicationContext）扫描到，否则该类中的配置信息就会被应用于所有的 @FeignClient 客户端(本例中 FooConfiguration 中的配置应该只对 StoreClient 起作用)。

**注意：** serviceId 属性已经被弃用了，取而代之的是 name 属性。

**注意：** 在先前的版本中在指定了 url 属性时 name 是可选属性，现在无论什么时候 name 都是必填属性。

name 和 url 属性也支持占位符：

```java
@FeignClient(name = "${feign.name}", url = "${feign.url}")
public interface StoreClient {
    //..
}
```

Spring Cloud Netflix 为 Feign 默认提供了以下的 beans：（BeanType beanName: ClassName）
* Decoder feignDecoder: ResponseEntityDecoder (which wraps a SpringDecoder)
* Encoder feignEncoder: SpringEncoder
* Logger feignLogger: Slf4jLogger
* Contract feignContract: SpringMvcContract
* Feign.Builder feignBuilder: HystrixFeign.Builder
* Client feignClient: 如果 Ribbon 在类路径中，并且启用了它，那么它就是 LoadBalancerFeignClient，否则，如果 Spring Cloud LoadBalancer 在类路径中，那么就使用 FeignBlockingLoadBalancerClient 。如果它们都不在类路径中，则使用默认的 Feign 客户端。

**注意：** spring-cloud-starter-openfeign 既包含 spring-cloud-starter-netflix-ribbon，也包含spring-cloud-starter-loadbalancer。

如果要使用 ApacheHttpClient，我们可以将 feign.httpclient.enabled 设置为 true，并保证类路径上有对应的库。还可以通过定义一个 org.apache.http.impl.client.CloseableHttpClient bean 来定制 HTTP 客户端。
如果要使用 OkHttpClient，我们可以将 feign.okhttp.enabled 设置为 true，并保证类路径上有对应的库。还可以通过定义一个 okhttp3.OkHttpClient bean 来定制 HTTP 客户端。

Spring Cloud Netflix 默认没有为 Feign 提供以下的 beans，但是在应用启动时依然会从上下文中查找这些 beans 来构造客户端对象：
* Logger.Level
* Retryer
* ErrorDecoder
* Request.Options
* `Collection<RequestInterceptor>`

如果想要覆盖 Spring Cloud Netflix 默认提供的 beans，需要在 @FeignClient 的 configuration 属性中指定一个配置类，并提供想要覆盖的 beans 即可：

```java
@Configuration
public class FooConfiguration {
    @Bean
    public Contract feignContract() {
        return new feign.Contract.Default();
    }

    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("user", "password");
    }
}
```

在本例中，我们用 feign.Contract.Default 代替了 SpringMvcContract, 并添加了一个 RequestInterceptor。以这种方式做的配置会在所有的 @FeignClient 中生效。

还可以使用配置属性配置 @FeignClient。

```yaml
feign:
  client:
    config:
      feignName:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full
        errorDecoder: com.example.SimpleErrorDecoder
        retryer: com.example.SimpleRetryer
        requestInterceptors:
          - com.example.FooRequestInterceptor
          - com.example.BarRequestInterceptor
        decode404: false
        encoder: com.example.SimpleEncoder
        decoder: com.example.SimpleDecoder
        contract: com.example.SimpleContract
```

可以在 @EnableFeignClients 的属性 defaultConfiguration 中以类似的方式指定默认配置，如上所述。不同的是，这种配置将适用于所有的 Feign 客户端。

如果我们想要使用配置属性来配置所有 @FeignClient，可以使用 default Feign 名称来创建配置属性。

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: basic
```

如果我们同时创建 @Configuration bean和配置属性，则配置属性优先级更高，它将覆盖 @Configuration 值。但是，如果我们想将更改为 @Configuration 优先级更高，则可以将 feign.client.default-to-properties 更改为 false。

**注意：** 如果我们需要在 RequestInterceptor 中使用 ThreadLocal 变量，那么我们需要将 Hystrix 的线程隔离策略设置为 SEMAPHORE，或者在 Feign 中禁用 Hystrix。

```yaml
# To disable Hystrix in Feign
feign:
  hystrix:
    enabled: false

# To set thread isolation to SEMAPHORE
hystrix:
  command:
    default:
      execution:
        isolation:
          strategy: SEMAPHORE
```

如果我们想创建多个具有相同名称或 url 的 Feign 客户端，以便它们指向相同的服务器，但每个服务器都具有不同的自定义配置，那么我们必须使用 @FeignClient 的 contextId 属性，以避免这些配置 bean 的名称冲突。

```java
@FeignClient(contextId = "fooClient", name = "stores", configuration = FooConfiguration.class)
public interface FooClient {
    //..
}
@FeignClient(contextId = "barClient", name = "stores", configuration = BarConfiguration.class)
public interface BarClient {
    //..
}
```

## Feign 对 Hystrix 的支持
如果 Hystrix 在类路径中且设置了 feign.hystrix.enabled=true，Feign 会默认将所有方法都封装到断路器中。还可以返回 com.netflix.hystrix.HystrixCommand。这允许我们使用响应模式（通过调用 .toObservable() 或 .observe() 或 异步使用（通过调用.queue()））。

如果想只关闭指定客户端的 Hystrix 支持，创建一个 Feign.Builder 组件并标注为 @Scope(prototype)：

```java
@Configuration
public class FooConfiguration {
    @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder() {
        return Feign.builder();
    }
}
```

## Feign 对 Hystrix Fallback 的支持
Hystrix 支持 fallback 的概念，即当断路器打开或发生错误时执行指定的失败逻辑。要为指定的 @FeignClient 启用 Fallback 支持， 需要在 fallback 性中指定实现类。还需要将实现类声明为 Spring bean。

```java
@FeignClient(name = "hello", fallback = HystrixClientFallback.class)
protected interface HystrixClient {
    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    Hello iFailSometimes();
}

static class HystrixClientFallback implements HystrixClient {
    @Override
    public Hello iFailSometimes() {
        return new Hello("fallback");
    }
}
```

如果需要访问 fallback 触发的原因，可以在 @FeignClient 中使用 fallbackFactory 属性。

```java
@FeignClient(name = "hello", fallbackFactory = HystrixClientFallbackFactory.class)
protected interface HystrixClient {
    @RequestMapping(method = RequestMethod.GET, value = "/hello")
    Hello iFailSometimes();
}

@Component
static class HystrixClientFallbackFactory implements FallbackFactory<HystrixClient> {
    @Override
    public HystrixClient create(Throwable cause) {
        return new HystrixClient() {
            @Override
            public Hello iFailSometimes() {
                return new Hello("fallback; reason was: " + cause.getMessage());
            }
        };
    }
}
```

**注意：** Feign 对 Hystrix Fallback 的支持有一个限制：对于返回 com.netflix.hystrix.HystrixCommand 或 rx.Observable 对象的方法，fallback 不起作用。

## Feign 和 @Primary
当使用 Feign 和 Hystrix fallbacks 时，ApplicationContext 会存在多个相同类型的 bean。这将导致 @Autowired 不能工作，因为没有一个精确的 bean，或者一个标记为主 bean。为了解决这个问题，Spring Cloud Netflix 将所有 Feign 实例标记为 @Primary，因此 Spring Framework 将知道注入哪个 bean。在某些情况下，这可能是不可取的。要关闭此行为，需将 @FeignClient 的 primary 属性设置为 false。

```java
@FeignClient(name = "hello", primary = false)
public interface HelloClient {
    // methods here
}
```

## Feign 对继承的支持
Feign 可以通过 Java 的接口支持继承。我们可以把一些公共的操作放到父接口中，然后定义子接口继承之：

```java
public interface UserService {

    @RequestMapping(method = RequestMethod.GET, value ="/users/{id}")
    User getUser(@PathVariable("id") long id);
}
```

```java
@RestController
public class UserResource implements UserService {

}
```

```java
package project.user;

@FeignClient("users")
public interface UserClient extends UserService {

}
```

**注意：在服务的调用端和提供端共用同一个接口定义是不明智的，这会将调用端和提供端的代码紧紧耦合在一起。同时在 SpringMVC 中会有问题，因为请求参数映射是不能被继承的。** 

## Feign 对压缩的支持
我们可能会想要对请求/响应数据进行 Gzip 压缩，指定以下参数即可：

```yaml
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```

也可以添加一些更细粒度的配置：

```yaml
feign.compression.request.enabled=true
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048
```

上面的 3 个参数可以让我们选择对哪种请求进行压缩，并设置一个最小请求大小的阀值。

对于除 OkHttpClient 外的 HTTP 客户端，可以启用默认的 gzip 解码器对 gzip 响应进行 UTF-8编码解码：

```yaml
feign.compression.response.enabled=true
feign.compression.response.useGzipDecoder=true
```

## Feign 日志
每一个 @FeignClient 都会创建一个 Logger, Logger 的名字就是接口的全限定名。Feign 的日志配置参数仅支持 DEBUG：

```yaml
logging.level.project.user.UserClient: DEBUG
```

Logger.Level 对象允许我们为指定客户端配置想记录哪些信息：
* NONE 不记录任何信息，默认值。
* BASIC 记录请求方法、请求URL、状态码和用时。
* HEADERS 在BASIC的基础上再记录一些常用信息。
* FULL 记录请求和响应报文的全部内容。

下面的示例将 Level 设置为 FULL：

```java
@Configuration
public class FooConfiguration {
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

## Feign 对 @QueryMap 的支持
OpenFeign @QueryMap 注解支持将 POJOs 作为 GET 参数映射。不幸的是，默认的 OpenFeign QueryMap 注解与 Spring 不兼容，因为它缺少值属性。

Spring Cloud OpenFeign 提供了一个等价的 @SpringQueryMap 注解，用于将 POJO 或 Map 参数注释为查询参数映射。

例如，Params 类定义了参数 param1 和 param2：

```java
// Params.java
public class Params {
    private String param1;
    private String param2;

    // [Getters and setters omitted for brevity]
}
```

下面的 Feign 客户端通过使用 @SpringQueryMap 注释来使用 Params类：

```java
@FeignClient("demo")
public interface DemoTemplate {

    @GetMapping(path = "/demo")
    String demoEndpoint(@SpringQueryMap Params params);
}
```

如果需要对生成的查询参数映射进行更多的控制，可以实现一个定制的 QueryMapEncoder bean。

## HATEOAS 支持
Spring 提供了一些 API 来创建遵循 HATEOAS 原则、Spring HATEOAS 和 Spring Data REST 的 REST 表示。

如果项目使用 org.springframework.boot:spring-boot-starter-hateoas starter 或 org.springframework.boot:spring-boot-starter-data-rest starter，默认启用 Feign HATEOAS 支持。

当 HATEOAS 支持被启用时，Feign 客户端可以序列化和反序列化 HATEOAS 表示模型：EntityModel、CollectionModel 和 PagedModel。

```java
@FeignClient("demo")
public interface DemoTemplate {

    @GetMapping(path = "/stores")
    CollectionModel<Store> getStores();
}
```

## 参考资料
1. [声明式REST客户端Feign](https://www.bookstack.cn/read/spring-cloud-docs/docs-user-guide-feign.md)