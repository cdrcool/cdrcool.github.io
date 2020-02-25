---
title: 后端框架-Swagger 接口文档
date: 2020-02-25 19:06:00
categories: 后端框架
tags:
    - Swagger
---
[Swagger](https://swagger.io/) 是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务。

利用[SpringFox](https://springfox.github.io/springfox/)，我们可以很快的将 Swagger 集成到 Spring Boot 项目中。

## Maven 依赖
```xml
<!-- api doc -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
</dependency>
```

## 启用 Swagger
```java
@EnableSwagger2
@EnableConfigurationProperties(SwaggerProperties.class)
@Configuration
public class SwaggerConfig {
    private final SwaggerProperties swaggerProperties;

    public SwaggerConfig(SwaggerProperties swaggerProperties) {
        this.swaggerProperties = swaggerProperties;
    }

    @Bean
    public Docket docket() {
        return new Docket(DocumentationType.SWAGGER_2).apiInfo(apiInfo());
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title(swaggerProperties.getTitle())
                .description(swaggerProperties.getDescription())
                .contact(contact())
                .version(swaggerProperties.getVersion())
                .build();
    }

    private Contact contact() {
        return new Contact(swaggerProperties.getContact().getName(),
                swaggerProperties.getContact().getUrl(),
                swaggerProperties.getContact().getEmail());
    }
}

@Data
@ConfigurationProperties(prefix = "swagger")
public class SwaggerProperties {

    /**
     * 标题
     */
    private String title;

    /**
     * 描述
     */
    private String description;

    /**
     * 版本号
     */
    private String version;

    /**
     * 联系人
     */
    private final Contact contact = new Contact();

    /**
     * 联系人
     */
    @Data
    public static class Contact {
        /**
         * 名字
         */
        private String name;

        /**
         * 主页
         */
        private String url;

        /**
         * 邮箱
         */
        private String email;
    }
}

@Configuration
public class WebMvcConfig extends WebMvcConfigurationSupport {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/**").addResourceLocations("classpath:/static/");
        registry.addResourceHandler("swagger-ui.html").addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**").addResourceLocations("classpath:/META-INF/resources/webjars/");
        super.addResourceHandlers(registry);
    }
}
```

**属性配置示例：**
```yaml
swagger:
  title: Spring Boot Samples
  description: Spring Boot Samples
  version: 1.0
  contact:
    name: cdrcool
    url: http://cdrcool.github.io/
    email: cdrcool@163.com
```

## Swagger 常用注解
* @Api 用于类，说明该类的作用
* @ApiOperation 用于方法，说明方法的用途
* @ApiImplicitParams 用于方法，包含一组参数说明
* @ApiImplicitParam 用于 @ApiImplicitParams 注解中，指定一个参数的配置信息
* @ApiResponses 用于方法，包含一组响应说明
* @ApiResponse 用于 @ApiResponses 中，指定一个响应的配置信息
* @ApiModel 用于类，对类进行说明
* @ApiModelProperty 用于类属性或类方法，对类属性进行说明

## 访问
应用启动后，访问 http://{ip}:{port}/{projectname}/swagger-ui.html 即可查看自动生成的 API 接口文档，JSON 数据可通过接口 http://{ip}:{port}/{projectname}/v2/api-docs 获取。