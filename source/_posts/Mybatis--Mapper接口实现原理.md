---
title: Mybatis--Mapper接口实现原理
date: 2019-09-17 16:48:25
categories: Mybatis
---
使用Mybatis注解进行开发时，dao的定义和引用都很简单，比如下面这样：
```java
@Repository
@Mapper
public interface CityMapper {

    @Select("SELECT * FROM city WHERE state = #{state}")
    City findByState(@Param("state") String state);
}

@SpringBootApplication
public class DemoApplication implements CommandLineRunner {
    private final CityMapper cityMapper;

    public DemoApplication(CityMapper cityMapper) {
        this.cityMapper = cityMapper;
    }
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

     @Override
    public void run(String... args) {
        System.out.println(this.cityMapper.findByState("CA"));
    }
}
```
那么Mapper接口的实现原理是什么呢？为什么配合使用Mybatis相关注解，只需要定义一个接口，就能实现bean的注册与依赖注入？

其实现原理可以简单概括为：
* Spring IoC容器在注册bean "cityMapper"时，其实际类型是 [**`FactoryBean`**](https://cdrcool.github.io/2019/09/13/Spring%20Framework--BeanFactory、FactoryBean、ObjectFactory)
* 从Spring IoC容器中获取bean "cityMapper"时，获取的不是`FactoryBean`本身，而是`FactoryBean`的`getObject`方法返回的对象
* 该对象是CityMapper接口的实现类，基于jdk动态代理技术实现

大概了解了原理之后，就开始分析源码吧。

## 注册
### MybatisAutoConfiguration
Spring Boot应用启动后，会扫描Mybatis自动装配类`MybatisAutoConfiguration`，该类中定义了内部类`AutoConfiguredMapperScannerRegistrar`和`MapperScannerRegistrarNotFoundConfiguration`，其中`MapperScannerRegistrarNotFoundConfiguration`是一个配置类（添加了@Configuration），同时它导入（@Import）了内部类`AutoConfiguredMapperScannerRegistrar`，而内部类`AutoConfiguredMapperScannerRegistrar`又实现了`ImportBeanDefinitionRegistrar`接口，于是在装配`AutoConfiguredMapperScannerRegistrar`的时候，`AutoConfiguredMapperScannerRegistrar`会调用方法`registerBeanDefinitions`往Spring IoC容器中注册自定义bean definitions，这里的bean是`MapperScannerConfigurer`。

```java
@org.springframework.context.annotation.Configuration
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, MybatisLanguageDriverAutoConfiguration.class })
public class MybatisAutoConfiguration implements InitializingBean {
    

    /**
    * This will just scan the same base package as Spring Boot does. If you want more power, you can explicitly use
    * {@link org.mybatis.spring.annotation.MapperScan} but this will get typed mappers working correctly, out-of-the-box,
    * similar to using Spring Data JPA repositories.
    */
    public static class AutoConfiguredMapperScannerRegistrar implements BeanFactoryAware, ImportBeanDefinitionRegistrar {
    
        private BeanFactory beanFactory;
        
        @Override
        public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        
            if (!AutoConfigurationPackages.has(this.beanFactory)) {
                logger.debug("Could not determine auto-configuration package, automatic mapper scanning disabled.");
                return;
            }
            
            logger.debug("Searching for mappers annotated with @Mapper");
            
            List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
            if (logger.isDebugEnabled()) {
                packages.forEach(pkg -> logger.debug("Using auto-configuration base package '{}'", pkg));
            }
            
            BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
            builder.addPropertyValue("processPropertyPlaceHolders", true);
            builder.addPropertyValue("annotationClass", Mapper.class);
            builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(packages));
            BeanWrapper beanWrapper = new BeanWrapperImpl(MapperScannerConfigurer.class);
            Stream.of(beanWrapper.getPropertyDescriptors())
                // Need to mybatis-spring 2.0.2+
                .filter(x -> x.getName().equals("lazyInitialization")).findAny()
                .ifPresent(x -> builder.addPropertyValue("lazyInitialization", "${mybatis.lazy-initialization:false}"));
            registry.registerBeanDefinition(MapperScannerConfigurer.class.getName(), builder.getBeanDefinition());
        }
    }

    ...
    
    /**
    * If mapper registering configuration or mapper scanning configuration not present, this configuration allow to scan
    * mappers based on the same component-scanning path as Spring Boot itself.
    */
    @org.springframework.context.annotation.Configuration
    @Import(AutoConfiguredMapperScannerRegistrar.class)
    @ConditionalOnMissingBean({ MapperFactoryBean.class, MapperScannerConfigurer.class })
    public static class MapperScannerRegistrarNotFoundConfiguration implements InitializingBean {
    
        @Override
        public void afterPropertiesSet() {
            logger.debug(
                "Not found configuration for registering mapper bean using @MapperScan, MapperFactoryBean and MapperScannerConfigurer.");
        }
    
    }

}
```

### MapperScannerConfigurer
`MapperScannerConfigurer`实现了`BeanDefinitionRegistryPostProcessor`接口，因此通过重写`postProcessBeanDefinitionRegistry`方法可以实现自定义的注册bean定义的逻辑。

```java
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {
    
    ...

    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        if (this.processPropertyPlaceHolders) {
            this.processPropertyPlaceHolders();
        }

        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
        scanner.setAddToConfig(this.addToConfig);
        scanner.setAnnotationClass(this.annotationClass);
        scanner.setMarkerInterface(this.markerInterface);
        scanner.setSqlSessionFactory(this.sqlSessionFactory);
        scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
        scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
        scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
        scanner.setResourceLoader(this.applicationContext);
        scanner.setBeanNameGenerator(this.nameGenerator);
        scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
        if (StringUtils.hasText(this.lazyInitialization)) {
            scanner.setLazyInitialization(Boolean.valueOf(this.lazyInitialization));
        }

        scanner.registerFilters();
        scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ",; \t\n"));
    }

    ...
}
```

### ClassPathMapperScanner
`ClassPathMapperScanner`继承了`ClassPathBeanDefinitionScanner`，`ClassPathBeanDefinitionScanner`是spring-context类库下的类，它用于扫描类路径下所有类，并将符合过滤条件的类注册到IOC容器内。
`ClassPathMapperScanner`会将扫描到的Mapper接口的BeanDefinition的bean class设置为`MapperFactoryBean`。

```java

public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {

    ...

    public Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
        if (beanDefinitions.isEmpty()) {
            LOGGER.warn(() -> {
                return "No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.";
            });
        } else {
            this.processBeanDefinitions(beanDefinitions);
        }

        return beanDefinitions;
    }

    private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
        GenericBeanDefinition definition;
        for(Iterator var3 = beanDefinitions.iterator(); var3.hasNext(); definition.setLazyInit(this.lazyInitialization)) {
            BeanDefinitionHolder holder = (BeanDefinitionHolder)var3.next();
            definition = (GenericBeanDefinition)holder.getBeanDefinition();
            String beanClassName = definition.getBeanClassName();
            LOGGER.debug(() -> {
                return "Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + beanClassName + "' mapperInterface";
            });
            definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName);
            // 关键：设置bean为MapperFactoryBean
            definition.setBeanClass(this.mapperFactoryBeanClass);
            definition.getPropertyValues().add("addToConfig", this.addToConfig);
            boolean explicitFactoryUsed = false;
            if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
                definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
                explicitFactoryUsed = true;
            } else if (this.sqlSessionFactory != null) {
                definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
                explicitFactoryUsed = true;
            }

            if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
                if (explicitFactoryUsed) {
                    LOGGER.warn(() -> {
                        return "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.";
                    });
                }

                definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
                explicitFactoryUsed = true;
            } else if (this.sqlSessionTemplate != null) {
                if (explicitFactoryUsed) {
                    LOGGER.warn(() -> {
                        return "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.";
                    });
                }

                definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
                explicitFactoryUsed = true;
            }

            if (!explicitFactoryUsed) {
                LOGGER.debug(() -> {
                    return "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.";
                });
                definition.setAutowireMode(2);
            }
        }

    }

    ...

}

public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {

    ...

	/**
	 * Perform a scan within the specified base packages.
	 * @param basePackages the packages to check for annotated classes
	 * @return number of beans registered
	 */
	public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

		doScan(basePackages);

		// Register annotation config processors, if necessary.
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}

	/**
	 * Perform a scan within the specified base packages,
	 * returning the registered bean definitions.
	 * <p>This method does <i>not</i> register an annotation config processor
	 * but rather leaves this up to the caller.
	 * @param basePackages the packages to check for annotated classes
	 * @return set of beans registered if any for tooling registration purposes (never {@code null})
	 */
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		for (String basePackage : basePackages) {
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}

    ...

}
```

### MapperFactoryBean
`MapperFactoryBean`是一个`FactoryBean`，定义了将从`SqlSessionTemplate`中获取实现了Mapper接口的真实bean。
另外，它还继承了`SqlSessionDaoSupport`，`SqlSessionDaoSupport`继承了`DaoSupport`，`DaoSupport`实现了`InitializingBean`，且在`afterPropertiesSet`中调用了`checkDaoConfig`，
而`checkDaoConfig`又会经Configuration.addMapper(Class<T> type)到调用MapperRegistry.addMapper(Class<T> type)，将Mapper Bean添加到`MapperRegistry`的注册表（knownMappers:Map<Class<?>, MapperProxyFactory<?>>）中。

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

    ...
    
    @Override
    public T getObject() throws Exception {
        return getSqlSession().getMapper(this.mapperInterface);
    }
    
    ...

    /**
     * bean实例化之后调用
     */
    @Override
    protected void checkDaoConfig() {
        super.checkDaoConfig();
        
        notNull(this.mapperInterface, "Property 'mapperInterface' is required");
        
        Configuration configuration = getSqlSession().getConfiguration();
        if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
            try {
                configuration.addMapper(this.mapperInterface);
            } catch (Exception e) {
                logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
                throw new IllegalArgumentException(e);
            } finally {
                ErrorContext.instance().reset();
            }
        }
    }
    
    ...

}
```

### MapperRegistry
MapperRegistry

```java
public class MapperRegistry {

    private final Configuration config;
    // 注册表，key:mapper interface，value: MapperProxyFactory
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
    
    public MapperRegistry(Configuration config) {
        this.config = config;
    }
    
    /**
     * 从注册表中获取MapperProxyFactory
     */
    @SuppressWarnings("unchecked")
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
            throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        }
        try {
            // 传入sqlSession，动态实例化type，即mapper interface
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
            throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
    }
    
    public <T> boolean hasMapper(Class<T> type) {
        return knownMappers.containsKey(type);
    }

    /**
     * 添加到注册表中
     */
    public <T> void addMapper(Class<T> type) {
        if (type.isInterface()) {
            if (hasMapper(type)) {
                throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
            }
            boolean loadCompleted = false;
            try {
                knownMappers.put(type, new MapperProxyFactory<>(type));
                // It's important that the type is added before the parser is run
                // otherwise the binding may automatically be attempted by the
                // mapper parser. If the type is already known, it won't try.
                MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
                parser.parse();
                loadCompleted = true;
            } finally {
                if (!loadCompleted) {
                    knownMappers.remove(type);
                }
            }
        }
    }
    
    /**
    * @since 3.2.2
    */
    public Collection<Class<?>> getMappers() {
        return Collections.unmodifiableCollection(knownMappers.keySet());
    }
    
    /**
    * @since 3.2.2
    */
    public void addMappers(String packageName, Class<?> superType) {
        ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
        resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
        Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
        for (Class<?> mapperClass : mapperSet) {
            addMapper(mapperClass);
        }
    }
    
    /**
    * @since 3.2.2
    */
    public void addMappers(String packageName) {
        addMappers(packageName, Object.class);
    }

}
```

## 查找
从Spring IoC容器中查找Mapper bean，首先会调用MapperFactoryBean.getObject()，然后经历：
MapperFactoryBean.getObject() -> 
SqlSessionManager.getMapper(Class<T> type) -> 
Configuration.getMapper(Class<T> type, SqlSession sqlSession) ->
MapperRegistry.getMapper(Class<T> type, SqlSession sqlSession) ->
一直到最终调用：MapperProxyFactory.newInstance(SqlSession sqlSession)。

### MapperProxyFactory
`MapperProxyFactory`是一个工厂类，用于动态创建代理类mapper proxy，也即mapper interface的实现类。

```java
public class MapperProxyFactory<T> {

    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();
    
    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }
    
    public Class<T> getMapperInterface() {
        return mapperInterface;
    }
    
    public Map<Method, MapperMethod> getMethodCache() {
        return methodCache;
    }
    
    @SuppressWarnings("unchecked")
    protected T newInstance(MapperProxy<T> mapperProxy) {
        // 利用jdk动态代理，动态创建实现类
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
    }
    
    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }

}
```
