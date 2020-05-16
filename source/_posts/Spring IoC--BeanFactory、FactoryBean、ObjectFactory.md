---
title: Spring IoC--BeanFactory、FactoryBean、ObjectFactory
date: 2019-09-13 11:26:48
categories: Spring IoC
---
## BeanFactory
Spring Ioc容器顶层接口。

![BeanFactory](/images/springframework/BeanFactory.png)

## FactoryBean
以Bean结尾，表示它是一个bean,但它不是普通的的bean，而是一个能生产对象的工厂bean，它的实现与设计模式中的工厂模式和修饰器模式类似。

![FactoryBean](/images/springframework/FactoryBean.png)

根据Bean的id从BeanFactory中获取的实际上是FactoryBean的getObject()方法返回的对象，而不是FactoryBean本身，如果要获取FactoryBean本身，需要在id前面加一个“&”符号。

```java
if (isFactoryBean(beanName)) {
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
        if (isEagerInit) {
            getBean(beanName);
        }
    }
}
```

## ObjectFactory
一个普通的对象工厂接口，它可以在调用时返回一个对象实例(可能是共享的或独立的)。

![ObjectFactory](/images/springframework/ObjectFactory.png)

通过查看Spring源码，可以发现SpringObjectFactory的应用之一就是:将创建对象的步骤封装到ObjectFactory中，交给自定义的Scope来选择是否需要创建对象来灵活的实现Scope。
* singleton
```java
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
```

* scope
```java
Object scopedInstance = scope.get(beanName, () -> {
    beforePrototypeCreation(beanName);
    try {
        return createBean(beanName, mbd, args);
    }
    finally {
        afterPrototypeCreation(beanName);
    }
});

public class ServletContextScope implements Scope, DisposableBean {
    ...

    public Object get(String name, ObjectFactory<?> objectFactory) {
        Object scopedObject = this.servletContext.getAttribute(name);
        if (scopedObject == null) {
            scopedObject = objectFactory.getObject();
            this.servletContext.setAttribute(name, scopedObject);
        }

        return scopedObject;
    }

    ...
}
```




