---
title: Spring Boot 集成 ZooKeeper
date: 2020-01-31 18:12:00
categories: ZooKeeper
tags:
    - Spring Boot
    - ZooKeeper
---
## Maven 依赖
```xml
<dependencies>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
    </dependency>
    <!-- zookeeper API的高层封装，大大简化 zookeeper 客户端编程，添加了例如 zookeeper 连接管理、重试机制等 -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
    </dependency>
    <!-- zookeeper 典型应用场景的实现，这些实现是基于 curator-framework -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
    </dependency>
</dependencies>
```

## 配置类
这里直接使用 curator 的 API：
```java
@Data
@Component
@ConfigurationProperties(prefix = "zookeeper")
public class ZooKeeperProperties {
    /**
     * ZooKeeper 服务器列表，由英文状态逗号分开的 host:port 字符串组成，每一个都代表一台 ZooKeeper 机器，例如，localhost:2181,localhost:2182,localhost:2183
     */
    private String connectString;

    /**
     * 会话超时时间，单位为毫秒。默认是 60000ms
     */
    private int sessionTimeout;

    /**
     * 连接创建超时时间，单位为毫秒，默认是 15000ms
     */
    private int connectionTimeout;

    /**
     * 已经重试的次数。如果是第一次重试，那么该参数为 0
     */
    private int retryCount;

    /**
     * 从第一次重试开始已经花费的时间，单位为毫秒
     */
    private int elapsedTime;
}

@Slf4j
@Configuration
public class ZooKeeperConfiguration {
    private final ZooKeeperProperties properties;

    public ZooKeeperConfiguration(ZooKeeperProperties properties) {
        this.properties = properties;
    }

    @Bean(initMethod = "start")
    public CuratorFramework curatorFramework() {
        return CuratorFrameworkFactory.newClient(
                properties.getConnectString(),
                properties.getSessionTimeout(),
                properties.getConnectionTimeout(),
                new RetryNTimes(properties.getRetryCount(), properties.getElapsedTime()));
    }
}
```

## 使用
```java
@Autowired
private CuratorFramework client;
```