---
title: Spring Boot--Spring Boot 应用监控 
date: 2021-06-16 10:56:16 
categories: Spring Boot
---

# 集成 Spring Boot Admin

## 创建 Spring Boot Admin Server
1. 创建新工程，引入以下依赖：

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
</dependency>
```

2. 在应用启动类上添加注解`@EnableAdminServer`，代码如下：
```java

@EnableAdminServer
@SpringBootApplication
public class MonitorApplication {

    public static void main(String[] args) {
        SpringApplication.run(MonitorApplication.class, args);
    }
}
```

3. `application.yml` 配置如下：
```yaml
server:
  port: 8081

spring:
  application:
    name: op-monitor
```

## 创建 Spring Boot Admin Client
1. 在 client 工程中引入以下依赖：
```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
</dependency>
```

2. 在`application.yml` 中添加以下配置：
```yaml
spring:
  application:
    name: op-admin
  boot:
    admin:
      client:
        url: http://localhost:8081

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always
    logfile:
      enabled: true
```

## Spring Boot Admin 监控示例
启动 server 与 client 两个工程之后，访问 http://localhost:8081/，即可看到以下信息：

![Spring Boot Admin 示例1.png](/images/springboot/SpringBootAdmin示例1.png)
![Spring Boot Admin 示例2.png](/images/springboot/SpringBootAdmin示例2.png)
![Spring Boot Admin 示例3.png](/images/springboot/SpringBootAdmin示例3.png)

# 集成 Prometheus

## Docker 容器
1. 编写配置文件
```yaml
scrape_configs:
  # 可随意指定
  - job_name: 'spring'
    # 多久采集一次数据
    scrape_interval: 15s
    # 采集时的超时时间
    scrape_timeout: 10s
    # 采集的路径
    metrics_path: '/actuator/prometheus'
    # 采集服务的地址，设置成 Spring Boot 应用所在服务器的具体地址
    static_configs:
      - targets: [ 'host.docker.internal:8080' ]
```

2. 拉取 Docker 镜像并创建容器
```cmd
docker run -d `
    --network op_net `
    --hostname prometheus `
    --name prometheus `
    -p 9090:9090 `
    -v /d/Workspace/IdeaProjects/onepiece/op-alliance/op-env:/etc/prometheus `
    prom/prometheus `
    --config.file=/etc/prometheus/prometheus.yml
```

## Spring Boot 工程
1. 在工程中引入以下依赖：

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

2. 开放 Prometheus 端点：
```yaml
management:
  endpoints:
    web:
      exposure:
        include: "prometheus"
```

## Prometheus 监控示例
启动 Docker 容器及 Spring Boot 工程之后，访问 http://localhost:9090/，即可看到以下信息：

![Prometheus 监控示例1.png](/images/springboot/Prometheus监控示例示例1.png)
![Prometheus 监控示例2.png](/images/springboot/Prometheus监控示例示例2.png)

# 集成 Grafana

## Docker 容器
拉取 Docker 镜像并创建容器：
```cmd
docker run -d `
    --network op_net `
    --hostname grafana `
    --name grafana `
    -p 3000:3000 `
    grafana/grafana
```

## Spring Boot 工程
添加以下配置属性：
```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
```

参考：[JVM (Micrometer)](https://grafana.com/grafana/dashboards/4701)

## Grafana 配置
1. 登录
启动 Docker 容器及 Spring Boot 工程之后，访问 http://localhost:3000/，即可看到登录界面：
![Grafana登录界面.png](/images/springboot/Grafana登录界面.png)

初始账号密码：admin/admin

2. 添加数据源
![Grafana添加数据源1.png](/images/springboot/Grafana添加数据源1.png)
![Grafana添加数据源2.png](/images/springboot/Grafana添加数据源2.png)

3. 下载 Dashboard Json
访问[Grafana Labs](https://grafana.com/grafana/dashboards)，搜索 **JVM (Micrometer)(4701)**，下载 Json 数据。
   
4. 导入 Dashboard Json
![Grafana 导入 Dashboard Json.png](/images/springboot/Grafana导入DashboardJson.png)

## 监控示例
![Grafana监控示例.png](/images/springboot/Grafana监控示例.png)

# 监控 Mysql
1. 拉取 mysqld-exporter镜像并启动容器
```cmd
docker run -d `
    --network op_net `
    --hostname mysqld-exporter `
    --hostname mysqld-exporter `
    -p 9104:9104 `
    -e DATA_SOURCE_NAME="root:root@(host.docker.internal:3306)/" `
    prom/mysqld-exporter
```

2. 补充 prometheus.yaml
```yaml
scrape_configs:
  - job_name: 'mysql'
    static_configs:
      - targets: ['host.docker.internal:9104']
        labels:
          instance: mysql
```

3. 创建 [MySQL Overview](https://grafana.com/grafana/dashboards/7362) Dashboard
