---
title: Spring IoC--Bean的初始化
date: 2019-09-13 13:46:02
categories: Spring IoC
---
Spring Boot是在刷新上下文（refreshContext）时执行单例Bean（non-lazy-init）的初始化的，其简要时序图如下：
![Bean初始化序列图](/images/springframework/Bean初始化序列图.png)

以上几个类之间的UML类图：
![Bean初始化类图](/images/springframework/Bean初始化类图.png)

## preInstantiateSingletons
流程图：
![Bean初始化](/images/springframework/Bean初始化.png)

源码：
```java
/**
 * Ensure that all non-lazy-init singletons are instantiated, also considering
 * {@link org.springframework.beans.factory.FactoryBean FactoryBeans}.
 * Typically invoked at the end of factory setup, if desired.
 * @throws BeansException if one of the singleton beans could not be created.
 * Note: This may have left the factory with some beans already initialized!
 * Call {@link #destroySingletons()} for full cleanup in this case.
 * @see #destroySingletons()
 *
 * 确保实例化了所有非惰性init单例，包括FactoryBean单例
 */
public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
        logger.trace("Pre-instantiating singletons in " + this);
    }

    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 如果不是抽象类，且是单例，且非延迟初始化
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 如果类型是FactoryBean
            if (isFactoryBean(beanName)) {
                // 对于FactoryBean，获取bean需要在前面添加“&”符号
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    final FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                        ((SmartFactoryBean<?>) factory)::isEagerInit,
                                getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    // 如果是立即初始化，就获取bean实例（这里的实例不是FactoryBean，而是FactoryBean的getObject()返回的bean）
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            }
            else {
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    // 为所有适用的bean触发初始化后回调
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```

`SmartInitializingSingleton`与`InitializingBean`的功能类似，都是在bean实例化后执行自定义初始化，不同的是它是在所有单例bean都创建之后才执行的。

## doGetBean
流程图：
![Bean创建](/images/springframework/Bean创建.png)

创建bean子流程图：
![创建Bean分支](/images/springframework/创建Bean分支.png)

源码：
```java
/**
 * Return an instance, which may be shared or independent, of the specified bean.
 * @param name the name of the bean to retrieve
 * @param requiredType the required type of the bean to retrieve
 * @param args arguments to use when creating a bean instance using explicit arguments
 * (only applied when creating a new instance as opposed to retrieving an existing one)
 * @param typeCheckOnly whether the instance is obtained for a type check,
 * not for actual use
 * @return an instance of the bean
 * @throws BeansException if the bean could not be created
 *
 * 返回指定bean的一个实例，该实例可以是共享的，也可以是独立的。
 */
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    final String beanName = transformedBeanName(name);
    Object bean;

    // Eagerly check singleton cache for manually registered singletons.
    // 先从缓存中获取bean
    Object sharedInstance = getSingleton(beanName);
    // 如果缓存中存在，且参数为null
    if (sharedInstance != null && args == null) {
        if (logger.isTraceEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        // 获取真正的bean
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        // 如果是原型bean且正在创建中，抛出异常
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // Check if bean definition exists in this factory.
        // 如果存在parent factory，且当前definitions中不包含beanName，就通过parent factory获取bean
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                        nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else if (requiredType != null) {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        if (!typeCheckOnly) {
            // 将指定的bean标记为已经创建或即将创建（alreadyCreated）
            markBeanAsCreated(beanName);
        }

        try {
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            // 保证当前bean所依赖的bean的初始化
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    // 为给定bean注册一个依赖bean，在销毁给定bean之前销毁它
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // Create bean instance.
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                // 从factoryBean中获取真正的bean实例
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    // 将原型注册为当前正在创建的状态（prototypesCurrentlyInCreation）
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    // 将原型标记为不在创建中
                    afterPrototypeCreation(beanName);
                }
                // 从factoryBean中获取真正的bean实例
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        // 将原型注册为当前正在创建的状态（prototypesCurrentlyInCreation）
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            // 将原型标记为不在创建中
                            afterPrototypeCreation(beanName);
                        }
                    });
                    // 从factoryBean中获取真正的bean实例
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                            "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // Check if required type matches the type of the actual bean instance.
    // 检查所需类型是否与实际bean实例的类型匹配
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        }
        catch (TypeMismatchException ex) {
            if (logger.isTraceEnabled()) {
                logger.trace("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

### getSingleton1
从一二三级缓存中获取bean并返回。

```java
/**
 * Return the (raw) singleton object registered under the given name.
 * <p>Checks already instantiated singletons and also allows for an early
 * reference to a currently created singleton (resolving a circular reference).
 * @param beanName the name of the bean to look for
 * @param allowEarlyReference whether early references should be created or not
 * @return the registered singleton object, or {@code null} if none found
 * 
 * 返回在给定名称下注册的(原始)单例对象
 * 检查已经实例化的单例，并允许对当前创建的单例进行早期引用(解析循环引用)
 */
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 从一级缓存中获取bean实例
    Object singletonObject = this.singletonObjects.get(beanName);
    // 如果一级缓存中不存在，且正在创建中（singletonsCurrentlyInCreation）
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) { // double-check:1
        synchronized (this.singletonObjects) {
            // 从二级缓存中获取early bean实例
            singletonObject = this.earlySingletonObjects.get(beanName);
            // 如果二级缓存中不存在，且允许创建early reference
            if (singletonObject == null && allowEarlyReference) {  // double-check:2
                // 从三级缓存中获取objectFactory对象
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                // 如果三级缓存中存在objectFactory对象
                if (singletonFactory != null) {
                    // 调用getObject返回真正的bean
                    singletonObject = singletonFactory.getObject();
                    // 添加到二级缓存
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 从三级缓存中移除
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

### getSingleton2
从一级缓存中获取bean，如果不存在就创建并注册一个新的bean。

```java
/**
 * Return the (raw) singleton object registered under the given name,
 * creating and registering a new one if none registered yet.
 * @param beanName the name of the bean
 * @param singletonFactory the ObjectFactory to lazily create the singleton
 * with, if necessary
 * @return the registered singleton object
 *
 * 返回在给定名称下注册的(原始)单例对象，如果还没有注册，则创建并注册一个新的单例对象。
 */
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        // 从一级缓存中获取bean实例
        Object singletonObject = this.singletonObjects.get(beanName);
        // 如果一级缓存中不存在
        if (singletonObject == null) {
            if (this.singletonsCurrentlyInDestruction) {
                throw new BeanCreationNotAllowedException(beanName,
                        "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                        "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            // 将正在创建的bean标记为正在创建中（singletonsCurrentlyInCreation）
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<>();
            }
            try {
                // 创建bean实例
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (IllegalStateException ex) {
                // Has the singleton object implicitly appeared in the meantime ->
                // if yes, proceed with it since the exception indicates that state.
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw ex;
                }
            }
            catch (BeanCreationException ex) {
                if (recordSuppressedExceptions) {
                    for (Exception suppressedException : this.suppressedExceptions) {
                        ex.addRelatedCause(suppressedException);
                    }
                }
                throw ex;
            }
            finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                // 移除正在创建中标记
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                // 将bean添加到一级缓存，同时从二级缓存、三级缓存中移除，并标记为已注册（registeredSingletons）
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```

### getObjectForBeanInstance
如果bean实例是FactoryBean，且要获取的不是它自身，就调用getObject()返回真正的bean，否则直接返回bean实例。

```java
/**
 * Get the object for the given bean instance, either the bean
 * instance itself or its created object in case of a FactoryBean.
 * @param beanInstance the shared bean instance
 * @param name name that may include factory dereference prefix
 * @param beanName the canonical bean name
 * @param mbd the merged bean definition
 * @return the object to expose for the bean
 *
 * 获取给定bean实例的对象，对于FactoryBean，要么是bean实例本身，要么是它创建的对象
 */
protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

    // Don't let calling code try to dereference the factory if the bean isn't a factory.
    // 如果想要获取FactoryBean本身,那么beanInstance必须是FactoryBean的实例
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
        }
    }

    // Now we have the bean instance, which may be a normal bean or a FactoryBean.
    // If it's a FactoryBean, we use it to create a bean instance, unless the
    // caller actually wants a reference to the factory.
    // 如果instance不是FactoryBean实例,或者想要获取的就是FactoryBean实例,那么直接返回就好
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
        return beanInstance;
    }

    Object object = null;
    if (mbd == null) {
        // 获取缓存的实例
        object = getCachedObjectForFactoryBean(beanName);
    }
    // 如果缓存中没有对象,那么从头准备bean defition实例化一个
    if (object == null) {
        // Return bean instance from factory.
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        // Caches object obtained from FactoryBean if it is a singleton.
        if (mbd == null && containsBeanDefinition(beanName)) {
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```

## createBean
```java
/**
 * Create a bean instance for the given merged bean definition (and arguments).
 * The bean definition will already have been merged with the parent definition
 * in case of a child definition.
 * <p>All bean retrieval methods delegate to this method for actual bean creation.
 *
 * Central method of this class: creates a bean instance,
 * populates the bean instance, applies post-processors, etc.
 *
 * @param beanName the name of the bean
 * @param mbd the merged bean definition for the bean
 * @param args explicit arguments to use for constructor or factory method invocation
 * @return a new instance of the bean
 * @throws BeanCreationException if the bean could not be created
 *
 * 为给定的合并bean定义(和参数)创建bean实例。对于子定义，bean定义已经与父定义合并。
 * 该类的核心方法:创建bean实例、填充bean实例、应用后处理程序等。
 */
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    if (logger.isTraceEnabled()) {
        logger.trace("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    // 确保此时bean类已实际解析：将bean类名解析为类引用(如果需要)，并将解析后的类存储在mbd（bean定义）中
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        // 克隆新的definition，并设置bean class
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
        // 验证并准备为该bean定义的方法覆盖
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        // 让beanpostprocessor有机会返回代理而不是目标bean实例
        // 当经过前置处理之后，返回的结果若不为空，那么会直接略过后续的Bean的创建而直接返回结果。AOP功能就是基于这里判断的.
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }

    try {
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isTraceEnabled()) {
            logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        // A previously detected exception with proper bean creation context already,
        // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}
```

## doCreateBean
```java
/**
 * Actually create the specified bean. Pre-creation processing has already happened
 * at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
 * <p>Differentiates between default bean instantiation, use of a
 * factory method, and autowiring a constructor.
 * @param beanName the name of the bean
 * @param mbd the merged bean definition for the bean
 * @param args explicit arguments to use for constructor or factory method invocation
 * @return a new instance of the bean
 * @throws BeanCreationException if the bean could not be created
 * @see #instantiateBean
 * @see #instantiateUsingFactoryMethod
 * @see #autowireConstructor
 *
 * 实际创建指定的bean。此时预创建处理已经完成，例如检查{@code postprocessbeforeinstance化}回调。
 * 默认bean实例化、使用工厂方法和自动装配构造函数之间有区别。
 */
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {

    // Instantiate the bean.
    // 包含了真正的bean对象和bean的class，以及PropertyDescriptor集合
    BeanWrapper instanceWrapper = null;
    // 单例的情况下尝试从factoryBeanInstanceCache获取instanceWrapper 
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    // 如果没有则需要自己创建
    if (instanceWrapper == null) {
        // 使用适当的实例化策略为指定的bean创建一个新实例:工厂方法、构造函数自动装配或简单实例化。
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    // 如果不是NullBean，则将resolvedTargetType属性设置为当前的WrappedClass
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // 遍历MergedBeanDefinitionPostProcessor接口的实现类，调用它们的postProcessMergedBeanDefinition方法，并将结果应用于bean definition
                // 像@Autowire、@Value、@Required、@PostConstruct、@PreDestor、@Scheduled等注解都是在这里预解析的
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    // 立即缓存单例，以便能够解析循环引用，即使是由生命周期接口(如BeanFactoryAware)触发的
    // 如果当前bean是单例，且支持循环依赖，且当前bean正在创建中，通过往singletonFactories添加一个objectFactory，这样后期如果有其他bean依赖该bean 可以从singletonFactories获取到bean，getEarlyBeanReference可以对返回的bean进行修改，这边目前除了可能会返回动态代理对象 其他的都是直接返回bean
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
            logger.trace("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        // 如果一级缓存中不存在，就添加到二级缓存，并从三级缓存中移除，同时标记为已注册（registeredSingletons）
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 填充bean实例，即完成属性注入
        populateBean(beanName, mbd, instanceWrapper);
        // 初始化bean
        // 1. 调用BeanNameAware、BeanClassLoaderAware、BeanFactoryAware接口方法
        // 2. 执行初始化之前的前置操作（BeanPostProcessor#getBeanPostProcessors()）
        // 3. 初始化：调用InitializingBean#afterPropertiesSet()，执行@PostConstruct方法
        // 4. 执行初始化之后的后置操作（BeanPostProcessor#postProcessAfterInitialization(result, beanName)）
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                            "Bean with name '" + beanName + "' has been injected into other beans [" +
                            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                            "] in its raw version as part of a circular reference, but has eventually been " +
                            "wrapped. This means that said other beans do not use the final version of the " +
                            "bean. This is often the result of over-eager type matching - consider using " +
                            "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // Register bean as disposable.
    // 注册disposable bean
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

## 循环依赖
对于单例Bean，如果是构造器循环依赖，Spring会抛出`BeanCurrentlyInCreationException`异常；如果是属性循环依赖，Spring则会利用三级缓存来解决。

假设有组件A、B相互依赖：
```java
@Component
public class ComponentA {
    @Autowired
    private  ComponentB componentB;
}

@Component
public class ComponentB {
    @Autowired
    private ComponentA componentA;
}
```

Spring会按以下步骤创建Bean A和Bean B：
1. 从缓存中获取Bean A，此时一、二、三级缓存中都还没有
2. 将Bean A标记为正在创建中
3. 将Bean A添加到三级缓存中，同时将其从二级缓存中移除，并将其标记为已注册
4. 填充Bean A，此时发现Bean A依赖于Bean B
5. 重复执行步骤1、2、3、4，不过Bean A与Bean B互换
6. 从缓存中获取Bean A，此时Bean A已存在于三级缓存中，就从三级缓存中获取并返回，同时将其添加到二级缓存中，并将其从三级缓存中移除
7. 创建Bean A完毕之后，清除其正在创建中的标记，然后将其添加到一级缓存中，同时将其从二、三级缓存中移除，并将其标记为已注册