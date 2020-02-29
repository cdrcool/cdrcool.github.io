---
title: Spring Cloud Consul 开发指南
date: 2020-02-26 15:21:00
categories: Spring Cloud
tags:
    - Spring Cloud Consul
---
## 安装 Consul
有关如何安装 Consul 的说明，参阅安装文档 [ installation documentation](https://www.consul.io/intro/getting-started/install.html)。

## Consul Agent
所有 Spring Cloud Consul 应用程序都必须有 Consul Agent。默认情况下，代理客户端位于 localhost:8500。有关如何启动代理客户端以及如何连接到 Consul 代理服务器群集的详细信息，参阅 [Agent documentation](https://consul.io/docs/agent/basics.html)。为了进行开发，安装 Consul 后，可以使用以下命令启动 Consul 代理：

```bash
./src/main/bash/local_run_consul.sh
```

这将在端口 8500上以服务器模式启动代理，并在 [http://localhost:8500/](http://localhost:8500/) 上提供用户界面。

## 服务发现与 Consul
服务发现是基于微服务架构的关键原则之一。尝试手动配置每个客户端或某种形式的约定可能非常困难，也可能非常脆弱。Consul 通过 HTTP API 和 DNS 提供服务发现服务。Spring Cloud Consul 利用 HTTP API 进行服务注册和发现。这并不妨碍非 Spring Cloud 应用程序利用 DNS 接口。Consul Agents 服务器在集群中运行，集群通过 gossip 协议进行通信，并使用 Raft 共识协议。

### 激活
要激活 Consul 服务发现，，需引入以下依赖：
                 
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

### 注册 与 Consul
当客户端向 Consul 注册时，它会提供有关自身的元数据，如主机和端口、id、名称和标签。默认情况下，会创建一个 HTTP 检查，Consul 每 10 秒访问一次 /health 端点。如果运行状况检查失败，则将服务实例标记为严重。

```java
@SpringBootApplication
@RestController
public class Application {

    @RequestMapping("/")
    public String home() {
        return "Hello world";
    }

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
```

（即完全正常的 Spring Boot 应用程序）。如果 Consul 客户端位于 localhost:8500 以外的其他位置，则需要配置来定位客户端。示例：

```yaml
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
```

**注意：** 如果使用 Spring Cloud Consul Config，则需要将上述值放在 bootstrap.yml 而不是 application.yml 中。

从环境中获取的默认服务名、实例 id 和端口分别是 ${spring.application.name}、Spring Context ID 和 ${server.port}。

要禁用 Consul Discovery客户端，可以将 spring.cloud.consul.discovery.enabled 设置为 false。当 spring.cloud.discovery.enabled 设置为 false 时，Consul Discovery Client 也将被禁用。

要禁用服务注册，可以将 spring.cloud.consul.discovery.register 设置为 false。

### HTTP 健康检查
Consul 实例的健康检查默认为 “/health”，这是 Spring Boot Actuator 应用程序中一个有用的端点的默认位置。如果使用非默认的上下文路径或 servlet 路径（例如 server.servletPath=/foo）或 management 端点路径（例如 management.server.servlet.context-path=/admin），则即使是 Actuator 应用程序，也需要更改这些内容。Consul 用来检查健康端点的时间间隔也可以配置。“10s” 和 "1min" 分别代表 10 秒和 1 分钟。示例：

```yaml
spring:
  cloud:
    consul:
      discovery:
        healthCheckPath: ${management.server.servlet.context-path}/health
        healthCheckInterval: 15s
```

可以通过设置 management.health.consun.enabled=false 来禁用运行状况检查。

#### 元数据和 Consul 标签
Consul 还不支持服务的元数据。Spring Cloud 的 ServiceInstance 有一个 Map<String，String> 元数据字段。在 Consul 正式支持元数据之前，Spring Cloud Consul 使用 Consul 标记来近似元数据。格式为 key=value 的标签将被拆分，并分别用作映射键和值。不带等号的标记将同时用作键和值。

```yaml
spring:
  cloud:
    consul:
      discovery:
        tags: foo=bar, baz
```

上面的配置将得到一个 foo→bar 和 baz→baz 的映射。

#### 使 Consul 实例 ID 唯一
默认情况下，Consul 实例注册的 ID 等于它的 Spring 应用程序上下文 ID。默认情况下，Spring 应用程序上下文 ID 是  ${spring.application.name}:comma,separated,profiles:${server.port}。对于大多数情况，这将允许一台机器上运行一个服务的多个实例。如果需要进一步的唯一性，通过使用 Spring Cloud，可以通过在 spring.cloud.consul.discovery.instanceId 中提供唯一的标识符来覆盖它。例如：

```yaml
spring:
  cloud:
    consul:
      discovery:
        instanceId: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
```

有了这个元数据和部署在 localhost 上的多个服务实例，随机值将进入其中，使实例唯一。在 Cloudfoundry 中，vcap.application.instance_id 将在 Spring 引导应用程序中自动填充，因此不需要随机值。

#### 在运行状况检查请求中应用 Headers
Headers 可以应用于健康检查请求。例如，如果要注册使用了 Vault Backend 的 Spring Cloud Config 服务：

```yaml
spring:
  cloud:
    consul:
      discovery:
        health-check-headers:
          X-Config-Token: 6442e58b-d1ea-182e-cfa5-cf9cddef0722
```

根据 HTTP 标准，每个 header 可以有多个值，在这种情况下，可以提供一个数组：

```yaml
spring:
  cloud:
    consul:
      discovery:
        health-check-headers:
          X-Config-Token:
            - "6442e58b-d1ea-182e-cfa5-cf9cddef0722"
            - "Some other value"
```

### 查找服务
#### 使用 Load-balancer
Spring Cloud 支持 Feign(一个 REST 客户端构建器)和 Spring RestTemplate，用于使用逻辑服务 names/id 而不是物理 URLs 查找服务。Feign 和 discovery-aware RestTemplate 都利用 Ribbon 来实现客户端负载平衡。

如果我们想使用 RestTemplate 访问服务 STORES，只需声明：

```java
@LoadBalanced
@Bean
public RestTemplate loadbalancedRestTemplate() {
     return new RestTemplate();
}
```

并像这样使用（注意我们如何使用 Consul 中的 STORES 服务 name/id 而不是完全限定的域名）：

```java
@Autowired
RestTemplate restTemplate;

public String getFirstProduct() {
   return this.restTemplate.getForObject("https://STORES/products/1", String.class);
}
```

如果我们在多个数据中心中有 Consul 集群，并且希望访问另一个数据中心中的服务，仅使用服务 name/id 是不够的。在这种情况下，使用属性 spring.cloud.consul.discovery.datacenters.STORES=dc-west，其中 STORES 是服务 name/id，dc-west 是 STORES 服务所在的数据中心。

Spring Cloud 现在还支持Spring Cloud LoadBalancer。

**注意：** 由于 Spring Cloud Ribbon 目前处于维护中，建议大家将 Spring.Cloud.loadbalancer.Ribbon.enabled 设置为 false，以便使用 BlockingLoadBalancerClient 而不是 RibbonLoadBalancerClient。

#### 使用 DiscoveryClient
我们还可以使用 org.springframework.cloud.client.discovery.DiscoveryClient，它为非特定于 Netflix 的发现客户端提供了一个简单的 API。

```java
@Autowired
private DiscoveryClient discoveryClient;

public String serviceUrl() {
    List<ServiceInstance> list = discoveryClient.getInstances("STORES");
    if (list != null && list.size() > 0 ) {
        return list.get(0).getUri();
    }
    return null;
}
```

### Consul Catalog Watch
Consul Catalog Watch 利用 Consul 的能力来 [watch services](https://www.consul.io/docs/agent/watches.html#services)。Catalog Watch 执行一个阻塞的 Consul HTTP API 调用，以确定是否有任何服务发生了变化。如果有新的服务数据，就会发布 Heartbeat Event。

可以通过调整 spring.cloud.consul.config.discovery.catalog-services-watch-delay 的值，来改变 Config Watch 调用的频率。默认值是 1000，以毫秒为单位。延迟是上一次调用结束和下一次调用开始后的时间量。

要禁用 Catalog Watch，设置 spring.cloud.consul.discovery.catalogServicesWatch.enabled=false。

该 watch 使用一个 Spring TaskScheduler 来调度与 Consul 的通话。默认情况下，它是一个 poolSize 为 1 的 ThreadPoolTaskScheduler。若要更改 TaskScheduler，需要创建一个类型为 TaskScheduler 的 bean，该 bean 的名称为 ConsulDiscoveryClientConfiguration.CATALOG_WATCH_TASK_SCHEDULER_NAME 常数。

## Consul 分布式配置
Consul 提供用于存储配置和其他元数据的 [Key/Value Store](https://consul.io/docs/agent/http/kv.html)。Spring Cloud Consul Config 是 [Config Server and Client](https://github.com/spring-cloud/spring-cloud-config) 的替代品。Configuration 是在特殊的“bootstrap”阶段被加载到Spring Environment 中。Configuration 默认存储在 /config 文件夹中。根据应用程序的名称和激活的 profiles 创建多个 PropertySource 实例，这些激活的 profiles 模仿 Spring Cloud Config 中解析属性的顺序。例如，一个名为“testApp”并带有“dev” profile 的应用程序将创建以下属性源：

```
config/testApp,dev/
config/testApp/
config/application,dev/
config/application/
```

最具体的属性源位于顶部，最不具体的位于底部。config/application 文件夹中的属性适用于使用 Consul 进行配置的所有应用程序。config/testApp 文件夹中的属性仅对名为“testApp”的服务实例可用。

当前 Configuration 是在应用程序启动时读取。向 /refresh 发送 HTTP POST 将导致重新加载配置。Config Watch 还将自动检测变化并重新加载应用程序上下文。

### 激活
要开始使用 Consul Configuration，需引入以下依赖：
 
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-config</artifactId>
</dependency>
```

这将启用 Spring Cloud Consul Config 的自动装配。

### 定制
Consul Config 可以使用以下属性自定义：

```yaml
spring:
  cloud:
    consul:
      config:
        enabled: true
        prefix: configuration
        defaultContext: apps
        profileSeparator: '::'
```

* enabled 将此值设置为 false 将禁用 Consul Config。
* prefix 设置配置值的基本文件夹。
* defaultContext 设置所有应用程序使用的文件夹名。
* profileSeparator 设置用于用 profiles 文件分隔属性源中的概要文件名称的分隔符的值

### Config Watch
Consul Config Watch 利用 Consul 的能力来 [watch a key prefix](https://www.consul.io/docs/agent/watches.html#keyprefix)。Config Watch 执行阻塞的 Consul HTTP API 调用，以确定当前应用程序的任何相关配置数据是否已更改。如果有新的配置数据，则会发布刷新事件。这相当于调用了 /refresh actuator 端点。

可以通过调整 spring.cloud.consul.config.watch.delay 的值，来改变 Config Watch 调用的频率。默认值是 1000，以毫秒为单位。延迟是上一次调用结束和下一次调用开始后的时间量。

要禁用 Config Watch，设置 spring.cloud.consul.config.watch.enabled=false。

该 watch 使用一个 Spring TaskScheduler 来调度与 Consul 的通话。默认情况下，它是一个 poolSize 为 1 的 ThreadPoolTaskScheduler。若要更改 TaskScheduler，需要创建一个类型为 TaskScheduler 的 bean，该 bean 的名称为 ConsulConfigAutoConfiguration.CONFIG_WATCH_TASK_SCHEDULER_NAME 常数。

### YAML 或 Properties 与配置
与单独的 key/value 对相比，以 YAML 或 Properties 格式存储配置属性可能更方便。设置 spring.cloud.consul.config.format 属性为 YAML 或 PROPERTIES。例如使用 YAML：

```yaml
spring:
  cloud:
    consul:
      config:
        format: YAML
```

YAML 必须在 Consul 的适当 data key 中设置。使用上面的默认 keys 看起来像：

```
config/testApp,dev/data
config/testApp/data
config/application,dev/data
config/application/data
```

我们可以在上面列出的任意 keys 中存储 YAML 文档。

我们可以使用 spring.cloud.consul.config.data-key 来更改 data key。

### git2consul 与配置
git2consul 是一个 Consul 社区项目，它将文件从 git 存储库加载到 Consul 中的各个密钥中。默认情况下，keys 的名称是文件的名称。YAML和 Properties 文件分别支持 .yml 和 .properties 文件扩展名。设置 spring.cloud.consul.config.format=FILES。例如：

```yaml
spring:
  cloud:
    consul:
      config:
        format: FILES
```

在 /config 中给定以下 keys，development profile 和 foo 的 application name：

```
.gitignore
application.yml
bar.properties
foo-development.properties
foo-production.yml
foo.properties
master.ref
```

将创建以下属性源：

```
config/foo-development.properties
config/foo.properties
config/application.yml
```

每个 key 的值必须是格式正确的 YAML 或 Properties 文件。

### 快速失败（Fail Fast）
在某些情况下（如本地开发或某些测试场景），如果 Consul 无法进行配置，则不失败可能很方便。在 bootstrap.yml 中设置 spring.cloud.consul.config.failFast=false 可以让配置模块记录一个警告，而不是抛出一个异常。这将允许应用程序继续正常启动。

### Consul 重试
如果我们预计在应用程序启动时 Consul 代理偶尔不可用，可以要求它在失败后继续尝试。我们需要将 spring-retry 和 spring-boot-starter-aop 添加到类路径中。默认行为是重试 6 次，初始回退间隔为 1000ms，后续回退的指数乘数为 1.1。我们可以使用 spring.cloud.consul.retry.* 配置属性来配置这些属性(以及其他属性)。这可以在 Spring Cloud Consul Config 和 Discovery registration 中使用。

要完全控制重试，可以添加一个类型为 RetryOperationsInterceptor、id 为 consulRetryInterceptor 的 @Bean。Spring Retry 提供了一个 RetryInterceptorBuilder，可以很容易地创建一个。

## Spring Cloud Bus 与 Consul
### 激活
要开始使用 Consul Bus，需引入以下依赖：
                 
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-bus</artifactId>
</dependency>
```

有关可用的 actuator 端点和如何发送自定义消息，参阅 [Spring Cloud Bus](https://cloud.spring.io/spring-cloud-bus/) 文档。

## 配置属性
要查看所有 Consul 相关配置属性的列表，请查看[附录页面](https://cloud.spring.io/spring-cloud-static/spring-cloud-consul/2.2.1.RELEASE/reference/html/appendix.html)。