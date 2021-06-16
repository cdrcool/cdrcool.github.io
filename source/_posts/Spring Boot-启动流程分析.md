---
title: Spring Boot--启动流程分析
date: 2019-09-12 19:52:04
categories: Spring Boot
---
启动Spring Boot应用只需执行以下代码：
```java
SpringApplication.run(ExampleApplication.class, args);
```
它会先初始化`SpringApplication`实例，然后执行该实例的`run`方法：
```java
new SpringApplication(primarySources).run(args);
```

## 初始化
```java
/**
 * 初始化SpringApplication实例
 *
 * @param resourceLoader 资源加载器，加载类路径或文件系统文件，默认为null
 * @param primarySources 应用程序主源，这里也就是ExampleApplication.class，application context将从此类开始加载beans
 */
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 推断应用程序类型，NONE|SERVLET|REACTIVE，一般为SERVLET
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 推断应用程序主类，这里也就是ExampleApplication.class
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

初始化比较简单：
1. 设置资源加载器，默认为null
2. 设置应用程序主源，一般为应用程序启动类
3. 推断应用程序类型，枚举值有：`NONE|SERVLET|REACTIVE`
4. 设置SpringApplication实例的initializers（`ApplicationContextInitializer`）属性
5. 设置SpringApplication实例的listeners（`ApplicationListener`）属性
6. 推断应用程序主类，其实就是启动类

### getSpringFactoriesInstances
在设置SpringApplication实例的的initializers和listeners属性时，调用了`getSpringFactoriesInstances`简单工厂方法，它是用来指定接口类型的示例集合。
```java
/**
 * 获取指定接口的实现类的实例集合
 *
 * @param type           接口class
 * @param parameterTypes 构造器参数类型数组，这里为空数组
 * @param args           构造器参数数组，这里为空数组
 * @return 实现类实例集合
 */
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    // 1. 如果指定了资源加载器，就获取资源加载其的类加载器，否则获取当前线程的类加载器
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    // 2. 获取指定接口的实现类的类名集合（全限定名，且去重）
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 3. 利用反射获取实例集合
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 4. 排序（添加了@Order注解或实现了Ordered接口）
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

### SpringFactoriesLoader
`SpringFactoriesLoader`是一个简单工厂类，它供Spring内部使用，用于读取**META-INF/spring.factories**文件（可能存在于类路径中的多个JAR文件中），返回指定接口的实现类的实例集合。

#### UML类图：
![SpringFactoriesLoader](/images/springboot/SpringFactoriesLoader.png)

#### 源码
重点看下`loadFactoryNames`方法：
```java
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    // 如果cache中存在，就直接返回
    // private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        // 如果类加载器不为空，就读取类路径下的目录“META-INF/spring.factories”，否则读取系统变量“META-INF/spring.factories”
        Enumeration<URL> urls = (classLoader != null ?
                classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryClassName = ((String) entry.getKey()).trim();
                for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryClassName, factoryName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

通过源码可以学习到`MultiValueMap`和`ConcurrentReferenceHashMap`的应用：
* MultiValueMap
同一个key，可以存在多个value
* ConcurrentReferenceHashMap
存储软引用或弱引用，内存溢出以前会清理该对象，适用于缓存

### 实现类
**spring-boot-version.jar**
```
# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
```

**spring-boot-autoconfigure-version.jar**
```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer
```

## 启动
```java
public ConfigurableApplicationContext run(String... args) {
    // 秒表计时器：统计代码段执行时间并一次输出
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // 设置系统变量：java.awt.headless
    configureHeadlessProperty();
    // 获取SpringApplicationRunListener实现类集合
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 组播starting事件
    listeners.starting();
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备环境 & 组播environmentPrepared事件（这个过程会加载application配置文件）
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);
        // 打印banner，可自定义
        Banner printedBanner = printBanner(environment);
        // 创建applicationContext（此时还只是空对象，对于web应用，其类型为AnnotationConfigServletWebServerApplicationContext）
        context = createApplicationContext();
        // 获取异常报告类数组，用于在发生异常时输出日志
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        // 准备上下文（组播contextPrepared -> 记录启动信息 -> contextLoaded事件）
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 刷新上下文
        refreshContext(context);
        // 刷新后操作
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            // 记录已启动 & 运行中 信息
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // 组播started事件
        listeners.started(context);
        // 调用其他runners的run方法（实现ApplicationRunner或CommandLineRunner接口）
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        // 处理异常（组播failed事件）
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // 组播running事件
        listeners.running(context);
    }
    catch (Throwable ex) {
        // 处理异常（组播failed事件）
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

### 流程图
![Spring Boot应用启动流程图](/images/springboot/Spring Boot应用启动流程图.png)
在启动过程中，如果发生异常，会执行异常处理，并组播failed事件。

### 实现类
**spring-boot-version.jar**
```
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener

# Error Reporters
org.springframework.boot.SpringBootExceptionReporter=\
org.springframework.boot.diagnostics.FailureAnalyzers
```