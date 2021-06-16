---
title: Spring Boot--事件监听机制
date: 2019-09-15 21:37:06
categories: Spring Boot
---
在[Spring Boot--启动流程分析](https://cdrcool.github.io/2019/09/12/Spring Boot--启动流程分析/)文中提到，在Spring Boot应用启动过程中，会经历某些固定的阶段，这些阶段中都会组播相应的事件，那么这些事件是如何发送，又是如何被接收处理的呢？本文将为大家详细介绍。

## 监听器模式
在介绍Spring Boot事件监听机制之前，我们有必要先了解事件监听器模式，事件监听器模式是观察者模式的另一种形态，同样基于事件驱动模型，事件监听器模式更加灵活，可以对不同事件作相应处理。其UML类图如下：
![事件监听器模式-简单类图](/images/springboot/事件监听器模式-简单.png)
由类图可知，事件监听器模式包含3中角色：
* 事件对象：java提供了标准事件对象类EventObject，它有一个属性source，source指数据源，表示是在哪个对象上发生的该事件。这里我们定义了CustomEvent子类,该类只接受指定的数据源对象EventSource。
* 事件监听类：同样，java提供了标准事件监听接口EventListener，它是一个空的接口。这里我们定义了CustomEventListener实现类，该类有一个方法onEvent，接受事件对象参数，表示对该事件进行处理。
* 事件源：可以是任意Object，通常会包含一个监听类集合，具有添加、删除事件监听类的行为，这里action操作会触发事件监听。

## 扩展：支持泛型
上面类图中，事件监听器只能监听某个具体事件，要使其能监听某类多个具体事件，可以采用泛型：
![事件监听器模式-泛型类图](/images/springboot/事件监听器模式-泛型.png)

## 扩展：组播类
像addEventListener、removeEventListener等操作，我们可以在单独的类中封装，然后当事件源中需要执行某方法时，将方法委托给该封装类处理，这样可以保证接口的一致性：
![事件组播](/images/springboot/事件组播.png)

## ApplicationListener
介绍完事件监听器模式之后，现在可以开始介绍Spring Boot事件监听机制啦（它可是采用标准的事件监听器模式实现的）。前文提到，Spring Boot应用启动过程中，会组播一系列的事件，这些事件继承于**SpringApplicationEvent**类：
![SpringApplicationEvent](/images/springboot/SpringApplicationEvent.png)
不难发现，这里的事件源对象正是SpringApplication类。有event，自然也有event listener：
![ApplicationListener](/images/springboot/ApplicationListener.png)
ApplicationListener有好多实现类，就不一一列举了，注意Spring Boot没有相应的定义SpringApplicationListener，我想是这是因为listener既会监听ApplicationEvent，也会监听SpringApplicationEvent。

## SpringApplicationRunListener
现在事件源、事件对象、事件对象监听类都清楚了，那么事件源是怎么触发事件监听的呢？下面贴出**SpringApplication.run**方法中的部分代码：
```
// 获取SpringApplicationRunListener实现类集合
SpringApplicationRunListeners listeners = getRunListeners(args);
// 组播starting事件
listeners.starting();
...
// 组播started事件
listeners.started(context);
...
// 组播running事件
listeners.running(context);
```
可以看到，事件的组播都是通过**SpringApplicationRunListeners**类实现的，之所以要创建该类是因为starting等操作都会触发一系列事件监听，所以它的构造函数接受**SpringApplicationRunListener**集合，看源码可知，除了failed方法，其他都是简单的循环调用单个listener的相应方法。
![SpringApplicationRunListener](/images/springboot/SpringApplicationRunListener.png)
再看**SpringApplicationRunListener**实现类，它有一个application属性，这就是事件源，然后还有一个initialMulticaster属性，还记得“扩展：组播类”吗，没错，**SimpleApplicationEventMulticaster**就是Spring Boot封装的组播类：
![ApplicationEventMulticaster](/images/springboot/ApplicationEventMulticaster.png)
其中，**ApplicationEventMulticaster**简单定义了增加、删除、组播等操作，**AbstractApplicationEventMulticaster**实现了增加、删除操作（组播操作留给子类实现），同时增加了两个supportsEvent重载方法，这两个方法是用于判断某listener是否支持相应指定的事件类型。
**SimpleApplicationEventMulticaster**主要实现组播了操作，它只会通知支持某指定事件类型的监听类，同时支持异常处理和异步操作。

总体UML类图（主要部分）：
![SpringBoot应用启动-事件监听机制](/images/springboot/SpringBoot应用启动-事件监听机制.png)
