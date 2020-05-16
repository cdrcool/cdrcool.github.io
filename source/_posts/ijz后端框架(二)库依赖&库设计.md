---
title: ijz后端框架(二)库依赖&库设计
date: 2019-01-24 21:43:48
categories: ijz后端框架
---
在库依赖&库设计方面，目前系统在四个方面存在不足：依赖库版本一致性、库与库松耦合、库内部高内聚、库依赖冗余。
## 依赖库版本一致性
### 意义
含义：多个库都依赖了某个库，被依赖的库在多个库中的版本号应该保持一致。

之所以强调一致性，是因为如果不一致，则在系统集成时很容易遇到版本冲突问题。
假设有以下场景：
* 库C有版本1和版本2
* 版本1中有类A，类A中方法X
* 版本2中新增了类B，且在类A中新增了方法Y，移除了方法X
* 库A依赖了库C的版本1
* 库B依赖了库C的版本2
* 项目M同时依赖了库A和库B

那么假如最终在项目M里：
* 只有库C的版本2，但某处调用了库C的版本1中类A的方法X，就会报方法X找不到异常
* 只有库C的版本1，但某处调用了库C的版本2中类A的方法Y，就会报方法Y找不到异常
* 只有库C的版本1，但某处引用了库C的版本2中类B，就会报类B找不到异常
* 同时存在库C的版本1和版本2，由于ClassLoader只会加载相同的class一次，因此也会出现以上问题

至于最终在项目M里采用的时哪个版本，则是根据以下规则：
* 声明优先原则
* 路径近者优先原则
* 排除原则
* 版本锁定原则

我们当然可以根据以上规则来限定最终依赖的版本，但是我们并不清楚会间接依赖到哪些冲突的库，即使知道了每个库都去显示声明也会很繁琐，而且当项目中同时调用了不同版本中互不兼容的方法时，还是会遇到问题。

### 现有问题
以资金项目为例，它依赖的很多库都存在多个版本，如下图所示：
![maven依赖版本不一致示例（一）](/images/ijz/maven依赖版本不一致示例（一）.png)
![maven依赖版本不一致示例（二）](/images/ijz/maven依赖版本不一致示例（二）.png)
![maven依赖版本不一致示例（三）](/images/ijz/maven依赖版本不一致示例（三）.png)
![maven依赖版本不一致示例（四）](/images/ijz/maven依赖版本不一致示例（四）.png)
![maven依赖版本不一致示例（五）](/images/ijz/maven依赖版本不一致示例（五）.png)
![maven依赖版本不一致示例（六）](/images/ijz/maven依赖版本不一致示例（六）.png)
![maven依赖版本不一致示例（七）](/images/ijz/maven依赖版本不一致示例（七）.png)

看到这有人会问了，不是版本不一致就会有问题吗，怎么这么多jar版本不一致，但却没发现啥问题呢？版本冲突问题我遇到过好几次了，印象较深的一次就是由于spring-data-jpa版本不一致而导致系统启动异常。现在之所以没出现问题，我想有两点原因：一是我们系统里很多jar虽然存在多个版本，但这些版本之间差别不大（通过版本号看出来），二是因为各版本间有冲突的类或方法我们并没有调用。

### 解决方案
项目中各依赖库版本不一致，根本原因在于没有全局定义库版本号。我们可以利用maven依赖传递机制，单据创建maven项目：[**ijz-boot-dependencies**](https://git.yonyouccs.com/ijz-modules-jars/ijz-boot-dependencies)，该项目中只有一个pom文件，所有可能被用到的库的版本号都预先定义在pom文件中，下面是pom文件具体内容：
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>2.1.3.RELEASE</version>
    </parent>

    <groupId>com.ijz.boot</groupId>
    <artifactId>ijz-boot-dependencies</artifactId>
    <version>1.0.0-RELEASE</version>
    <packaging>pom</packaging>

    <properties>
        <!-- ijz starters -->
        <dubbox-spring-boot-starter.version>1.0.0-RELEASE</dubbox-spring-boot-starter.version>
        <bill-spring-boot-starter.version>1.0.0-RELEASE</bill-spring-boot-starter.version>
        <shiro-spring-boot-starter.version>1.0.0-RELEASE</shiro-spring-boot-starter.version>

        <!-- ijz framework -->
        <ijz-export.version>1.0.0-RELEASE</ijz-export.version>

        <!-- icop -->
        <icop-config.version>1.5.0-RELEASE</icop-config.version>
        <icop-monitor-client.version>1.5.1-RELEASE</icop-monitor-client.version>
        <logstash-logback-encoder.version>5.3</logstash-logback-encoder.version>

        <!-- icop api -->
        <icop-orgcenter-api.version>1.5.19</icop-orgcenter-api.version>
        <icop-share-api.version>1.9.17.4-SNAPSHOT</icop-share-api.version>
        <icop-support-api.version>1.5.1-RELEASE</icop-support-api.version>
        <icop-file-api.version>1.5.2-RELEASE</icop-file-api.version>
        <icop-myproject-api.version>0.1.0-SNAPSHOT</icop-myproject-api.version>

        <!-- ijz api -->
        <ijz-contract-api.version>0.0.1-SNAPSHOT</ijz-contract-api.version>
        <ijz-cost-api.version>0.0.1-SNAPSHOT</ijz-cost-api.version>
        <ijz-finance-api.version>1.0.23-SNAPSHOT</ijz-finance-api.version>
        <ijz-tax-api.version>1.0.21-SNAPSHOT</ijz-tax-api.version>
        <ijz-budget-api.version>1.0.7-SNAPSHOT</ijz-budget-api.version>
        <ijz-material-api.version>0.0.1-SNAPSHOT</ijz-material-api.version>

        <!-- third party dependencies -->
        <!-- dubbo相关 -->
        <dubbo.version>2.8.4</dubbo.version>
        <io-netty.version>3.10.6.Final</io-netty.version>
        <zkclient.version>0.1</zkclient.version>

        <servlet-api.version>4.0.1</servlet-api.version>
        <springfox.verdion>2.9.2</springfox.verdion>
        <fastjson.version>1.2.56</fastjson.version>

        <!-- poi相关 -->
        <poi.version>4.0.1</poi.version>
        <jxls.version>2.5.1</jxls.version>
        <jxls-poi.version>1.1.0</jxls-poi.version>

        <!-- shiro相关 -->
        <shiro-spring.version>1.4.0</shiro-spring.version>
        <iuap-auth.version>1.5.5</iuap-auth.version>
        <iuap-utils.version>3.1.0-RELEASE</iuap-utils.version>
        <icop-core.version>1.5.9</icop-core.version>
        <icop-context.version>1.5.17</icop-context.version>
        <esapi.version>2.1.0.1</esapi.version>

        <!-- project -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>

        <!-- plugins -->
        <maven-compiler-plugin.version>3.8.0</maven-compiler-plugin.version>
        <maven-war-plugin.version>3.2.2</maven-war-plugin.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.ijz.spring.boot</groupId>
                <artifactId>dubbox-spring-boot-starter</artifactId>
                <version>${dubbox-spring-boot-starter.version}</version>
            </dependency>
            <dependency>
                <groupId>com.ijz.spring.boot</groupId>
                <artifactId>bill-spring-boot-starter</artifactId>
                <version>${bill-spring-boot-starter.version}</version>
            </dependency>
            <dependency>
                <groupId>com.ijz.spring.boot</groupId>
                <artifactId>shiro-spring-boot-starter</artifactId>
                <version>${shiro-spring-boot-starter.version}</version>
            </dependency>

            <dependency>
                <groupId>com.ijz.framework</groupId>
                <artifactId>ijz-export</artifactId>
                <version>${ijz-export.version}</version>
            </dependency>

            <dependency>
                <groupId>com.yonyou.construction.icop</groupId>
                <artifactId>icop-config</artifactId>
                <version>${icop-config.version}</version>
            </dependency>
            <dependency>
                <groupId>com.yonyou.construction.icop</groupId>
                <artifactId>icop-monitor-client</artifactId>
                <version>${icop-monitor-client.version}</version>
            </dependency>
            <dependency>
                <groupId>net.logstash.logback</groupId>
                <artifactId>logstash-logback-encoder</artifactId>
                <version>${logstash-logback-encoder.version}</version>
            </dependency>

            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>icop-orgcenter-api</artifactId>
                <version>${icop-orgcenter-api.version}</version>
                <exclusions>
                    <exclusion>
                        <groupId>net.logstash.logback</groupId>
                        <artifactId>logstash-logback-encoder</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>icop-share-api</artifactId>
                <version>${icop-share-api.version}</version>
                <exclusions>
                    <exclusion>
                        <groupId>net.logstash.logback</groupId>
                        <artifactId>logstash-logback-encoder</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.yyjz</groupId>
                        <artifactId>icop-exception</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>icop-support-api</artifactId>
                <version>${icop-support-api.version}</version>
                <exclusions>
                    <exclusion>
                        <groupId>net.logstash.logback</groupId>
                        <artifactId>logstash-logback-encoder</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.yyjz</groupId>
                        <artifactId>icop-exception</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>icop-file-api</artifactId>
                <version>${icop-file-api.version}</version>
                <exclusions>
                    <exclusion>
                        <groupId>commons-codec</groupId>
                        <artifactId>commons-codec</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>com.yyjz.icop</groupId>
                <artifactId>icop-myproject-api</artifactId>
                <version>${icop-myproject-api.version}</version>
            </dependency>

            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>ijz-contract-api</artifactId>
                <version>${ijz-contract-api.version}</version>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>ijz-cost-api</artifactId>
                <version>${ijz-cost-api.version}</version>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>ijz-finance-api</artifactId>
                <version>${ijz-finance-api.version}</version>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>ijz-tax-api</artifactId>
                <version>${ijz-tax-api.version}</version>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>ijz-budget-api</artifactId>
                <version>${ijz-budget-api.version}</version>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>ijz-material-api</artifactId>
                <version>${ijz-material-api.version}</version>
            </dependency>

            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>dubbo</artifactId>
                <version>${dubbo.version}</version>
            </dependency>
            <dependency>
                <groupId>io.netty</groupId>
                <artifactId>netty</artifactId>
                <version>${io-netty.version}</version>
            </dependency>
            <dependency>
                <groupId>com.github.sgroschupf</groupId>
                <artifactId>zkclient</artifactId>
                <version>${zkclient.version}</version>
            </dependency>

            <dependency>
                <groupId>javax.servlet</groupId>
                <artifactId>javax.servlet-api</artifactId>
                <version>${servlet-api.version}</version>
            </dependency>
            <dependency>
                <groupId>io.springfox</groupId>
                <artifactId>springfox-swagger2</artifactId>
                <version>${springfox.verdion}</version>
            </dependency>
            <dependency>
                <groupId>io.springfox</groupId>
                <artifactId>springfox-swagger-ui</artifactId>
                <version>${springfox.verdion}</version>
            </dependency>
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>fastjson</artifactId>
                <version>${fastjson.version}</version>
            </dependency>
            <dependency>
                <groupId>org.apache.poi</groupId>
                <artifactId>poi</artifactId>
                <version>${poi.version}</version>
            </dependency>
            <dependency>
                <groupId>org.apache.poi</groupId>
                <artifactId>poi-ooxml</artifactId>
                <version>${poi.version}</version>
            </dependency>
            <dependency>
                <groupId>org.jxls</groupId>
                <artifactId>jxls</artifactId>
                <version>${jxls.version}</version>
            </dependency>
            <dependency>
                <groupId>org.jxls</groupId>
                <artifactId>jxls-poi</artifactId>
                <version>${jxls-poi.version}</version>
            </dependency>

            <dependency>
                <groupId>org.apache.shiro</groupId>
                <artifactId>shiro-spring</artifactId>
                <version>${shiro-spring.version}</version>
            </dependency>
            <dependency>
                <groupId>com.yonyou.iuap</groupId>
                <artifactId>iuap-auth</artifactId>
                <version>${iuap-auth.version}</version>
            </dependency>
            <dependency>
                <groupId>com.yonyou.iuap</groupId>
                <artifactId>iuap-utils</artifactId>
                <version>${iuap-utils.version}</version>
                <exclusions>
                    <exclusion>
                        <groupId>commons-fileupload</groupId>
                        <artifactId>commons-fileupload</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>icop-core</artifactId>
                <version>${icop-core.version}</version>
                <!-- json-lib和spring-webmvc（这个也应该排除，但是没有spring-web）之外，其他都排除掉 -->
                <exclusions>
                    <exclusion>
                        <groupId>org.apache.commons</groupId>
                        <artifactId>commons-lang3</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.google.guava</groupId>
                        <artifactId>guava</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.apache.httpcomponents</groupId>
                        <artifactId>httpclient</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.apache.httpcomponents</groupId>
                        <artifactId>httpmime</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>commons-fileupload</groupId>
                        <artifactId>commons-fileupload</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.springframework</groupId>
                        <artifactId>spring-context-support</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.hibernate</groupId>
                        <artifactId>hibernate-entitymanager</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.springframework.data</groupId>
                        <artifactId>spring-data-jpa</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.freemarker</groupId>
                        <artifactId>freemarker</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>javax.servlet</groupId>
                        <artifactId>jstl</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.apache.shiro</groupId>
                        <artifactId>shiro-spring</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.apache.shiro</groupId>
                        <artifactId>shiro-ehcache</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>commons-codec</groupId>
                        <artifactId>commons-codec</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.owasp.esapi</groupId>
                        <artifactId>esapi</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.alibaba</groupId>
                        <artifactId>fastjson</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.fasterxml.jackson.core</groupId>
                        <artifactId>jackson-databind</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.fasterxml.jackson.module</groupId>
                        <artifactId>jackson-module-jaxb-annotations</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.json</groupId>
                        <artifactId>json</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.logback-extensions</groupId>
                        <artifactId>logback-ext-spring</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.slf4j</groupId>
                        <artifactId>slf4j-api</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>ch.qos.logback</groupId>
                        <artifactId>logback-classic</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.slf4j</groupId>
                        <artifactId>log4j-over-slf4j</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.slf4j</groupId>
                        <artifactId>jcl-over-slf4j</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.slf4j</groupId>
                        <artifactId>jul-to-slf4j</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.googlecode.log4jdbc</groupId>
                        <artifactId>log4jdbc</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>redis.clients</groupId>
                        <artifactId>jedis</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>io.springside</groupId>
                        <artifactId>springside-redis</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.yonyou.iuap</groupId>
                        <artifactId>iuap-securitylog-rest-sdk</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>junit</groupId>
                        <artifactId>junit</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.assertj</groupId>
                        <artifactId>assertj-core</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.springframework</groupId>
                        <artifactId>spring-test</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.eclipse.jetty.aggregate</groupId>
                        <artifactId>jetty-webapp</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.eclipse.jetty</groupId>
                        <artifactId>jetty-jsp</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.csource</groupId>
                        <artifactId>fastdfs_client</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>io.springside</groupId>
                        <artifactId>springside-core</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.apache.httpcomponents</groupId>
                        <artifactId>fluent-hc</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.yonyou.iuap</groupId>
                        <artifactId>iuap-auth</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.yyjz</groupId>
                        <artifactId>icop-database</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>commons-dbcp</groupId>
                        <artifactId>commons-dbcp</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.yyjz</groupId>
                        <artifactId>icop-dubbo-filter</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>net.logstash.logback</groupId>
                        <artifactId>logstash-logback-encoder</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>com.yyjz</groupId>
                <artifactId>icop-context</artifactId>
                <version>${icop-context.version}</version>
                <exclusions>
                    <exclusion>
                        <groupId>log4j</groupId>
                        <artifactId>log4j</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>com.yyjz</groupId>
                        <artifactId>icop-cache</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>org.owasp.esapi</groupId>
                <artifactId>esapi</artifactId>
                <version>${esapi.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <encoding>${project.build.sourceEncoding}</encoding>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>${maven-war-plugin.version}</version>
            </plugin>
        </plugins>
    </build>

    <!-- 从maven 私服拉取库 -->
    <repositories>
        <repository>
            <id>icop-nexus</id>
            <url>https://maven.yonyouccs.com/nexus/content/groups/public/</url>
            <snapshots>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
    </repositories>

    <!-- 发布到maven私服 -->
    <distributionManagement>
        <repository>
            <id>icop-release</id>
            <url>https://maven.yonyouccs.com/nexus/content/repositories/icop-release</url>
        </repository>
        <snapshotRepository>
            <id>icop-snapshot</id>
            <url>https://maven.yonyouccs.com/nexus/content/repositories/icop-snapshot</url>
        </snapshotRepository>
    </distributionManagement>
</project>
```
细心的朋友会发现，该项目继承自**spring-boot-dependencies**，一是为了避免重复定义大量第三方库的版本号，二来与spring boot保持一致，因为后端框架就是基于Spring Boot。

其他项目引入有以下两种方式引入ijz-boot-dependencies：
* 继承
```xml
<parent>
    <groupId>com.ijz.boot</groupId>
    <artifactId>ijz-boot-dependencies</artifactId>
    <version>1.0.0-RELEASE</version>
</parent>
```
* import
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.ijz.boot</groupId>
            <artifactId>ijz-boot-dependencies</artifactId>
            <version>1.0.0-RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
引入之后，项目中就不需要再指定各个库的版本号了：
```xml
<dependencies>
    <!-- 依赖的某个库 -->
    <dependency>
        <groupId>xx</groupId>
        <artifactId>xx</artifactId>
        <!-- <version>x.x</version> --> <!-- 不需要再指定版本号了 -->
    </dependency>
</dependencies>
```

### 适配icop
由于目前所有web项目都需要继承icop-parent-pom，所以只能采用import方式引入，于是我们可以再创建maven项目：[ijz-boot-starter-parent](https://git.yonyouccs.com/ijz-modules-jars/ijz-boot-starter-parent)，该项目同样只有一个pom文件，这个pom既继承了icop-parent-pom，又依赖了ijz-boot-dependencies，下面是pom具体内容：
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.yonyouccs</groupId>
        <artifactId>icop-parent-pom</artifactId>
        <version>1.6.2</version>
    </parent>

    <groupId>com.ijz.boot</groupId>
    <artifactId>ijz-boot-starter-parent</artifactId>
    <version>1.0.0-RELEASE</version>
    <packaging>pom</packaging>

    <properties>
        <!-- 覆盖父类中默认版本号 -->
        <icop-config.version>1.5.0-RELEASE</icop-config.version>
        <logback.version>1.2.3</logback.version>
        <jackson.version>2.9.8</jackson.version>
        <javassist.version>3.24.1-GA</javassist.version>
    </properties>

    <dependencies>
        <!-- 覆盖父类中定义的版本 -->
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.ijz.boot</groupId>
                <artifactId>ijz-boot-dependencies</artifactId>
                <version>1.0.0-RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- 覆盖父类中默认版本号 -->
            <dependency>
                <groupId>com.yonyou.construction.icop</groupId>
                <artifactId>icop-config</artifactId>
                <version>${icop-config.version}</version>
            </dependency>
            <dependency>
                <groupId>ch.qos.logback</groupId>
                <artifactId>logback-classic</artifactId>
                <version>${logback.version}</version>
            </dependency>
            <dependency>
                <groupId>com.fasterxml.jackson.core</groupId>
                <artifactId>jackson-databind</artifactId>
                <version>${jackson.version}</version>
            </dependency>
            <dependency>
                <groupId>org.javassist</groupId>
                <artifactId>javassist</artifactId>
                <version>${javassist.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- 从maven 私服拉取库 -->
    <repositories>
        <repository>
            <id>icop-nexus</id>
            <url>https://maven.yonyouccs.com/nexus/content/groups/public/</url>
            <snapshots>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
    </repositories>
</project>
```
如此一来，web项目只需继承**ijz-boot-starter-parent**，便既兼容icop，又引入了版本一致性。（库还是继承**ijz-boot-dependencies**）

## 库与库松耦合
### 意义
含义：库与库之间，尽量使其独立存在。

模块与模块之间依赖过多，会带来维护和扩展上的不便。例如模块A和模块B都直接依赖模块C，这样当模块C发生改变时，模块A和模块B可能都需要同步修改。如果修改时没有理清耦合关系，那么带来的后果可能是灾难性的。
那要怎么减小模块间的耦合呢？面向接口编程！例如下图，电器与插座之间是低耦合的关系，就算替换了不同的插座，电器依然可以正常的工作，因为电器都实现了相同的接口。
![低耦合示例](/images/ijz/低耦合示例.png)

### 现有问题
目前好多类库耦合都过于紧密。例如bpm回调过程中需要回写审批人，审批人是直接调用icop-context库中的的UserContext类获取的，这样就直接依赖了icop-context，如果后面我们更换了上下文库，又或icop-context库自身做了修改，那么bpm回调代码也需要做相应的更改。

### 解决方案
耦合问题不像版本一致性，它不是改下pom文件就能解决的，而是需要我们在类库设计时具体分析。以获取上下文为例，我们可以参考spring-data-jpa的设计思路：
spring-data-jpa提供了审计功能，如果我们在实体字段上添加了@CreatedBy或@LastModifiedBy注解，那么当实体被创建或修改时，就会自动将相应字段赋值为当前用户。但spring-data-jpa并没有依赖spring-security库来获取当前用户信息，而是提供了AuditorAware接口，用户信息是由我们在该接口的getCurrentAuditor方法中返回的，这样就达到了和spring-security库解耦的目的。

## 库内部高内聚
### 意义
含义：封装良好的类库应该恰如其分的只提供一类功能。

在6大设计原则中有**单一职责**：一个类，应该只有一个引起它变化的原因。类如此，库也应如此，可以理解为类是最小的库，库是大的类群。遵循该原则可以使库的复杂度降低，可读性提高，变更引起的风险降低。

### 现有问题
库icop-pubapp-platform，它里面即包含持久化，又包含导入导出和打印，还做了es数据同步等。

### 解决方案
拆分，比如上面的icop-pubapp-platform，就可以拆分成4个：持久化、导入导出、打印、es数据同步。

## 库依赖冗余
### 意义
含义：一个库不应该依赖它不需要的库，库的依赖应尽量的少。

如果库过于臃肿，不仅会导致应用启动时间加长，也会带来潜在的风险。

### 现有问题
最典型的就是库icop-core，它的设计初衷是作为一个基础库供其它项目使用，基础库应该尽量简洁，只封装一些核心的功能，但是它却引入了大量没用到的库：
![依赖库（一）](/images/ijz/icop-core依赖库（一）.png)
![依赖库（二）](/images/ijz/icop-core依赖库（二）.png)
再就是一些api库，本应是很简单的只提供接口，却要引入一堆库：
![icop-myproject-api依赖库](/images/ijz/icop-myproject-api依赖库.png)

### 解决方案
对于没有用到的依赖，移除掉。