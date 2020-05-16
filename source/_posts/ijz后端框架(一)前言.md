---
title: ijz后端框架(一)前言
date: 2019-02-23 17:09:31
categories: ijz后端框架
---
之前我有反映目前后端框架中存在哪些问题，但没有系统的梳理；也有表示自己在基于Spring Boot对后端框架做改造，但也没有具体的展示。本文就是简单的概要我认为后端框架中有哪些不足之处，做了哪些扩展，又为什么要引入Spring Boot，至于更详细的阐述，将在后续的文章中陆续展开。

## 不足之处
* xml配置繁琐
相信大家在创建新web项目的时候，都会从其它项目中拷贝applicationContext.xml、applicationContext-dubbo.xml和applicationContext-security.xml等配置文件，这些配置文件都是必须的，但里面的内容除了数据库名、web context等地方外，绝大部分都是我们不需要修改的。基于Spring Boot,我们可以引入或定制各类即插即用的starters，再也不需要配置xml！
* 依赖库版本不一致
在我们应用中，同一个第三方库可能有好几个版本，这样很容易出现版本冲突问题，原因在于缺少全局性的pom文件来统一指定版本号。
* 库与库紧耦合
库与库耦合度过高，就可能会出现了某个库做了修改，导致其它依赖了它的库都要做相应的更改，这类问题存在于我们好多类库中。
比如我们有个excel导入导出的库icop-pubapp-excel，它里面依赖了另外两个库：icop-database和icop-refer，这是必要的吗？
之所以会用到icop-database，只是因为用到了它里面的异常工具类ExceptionUtils，且不说这个类是否有必要，我们至于因为一个异常类就引入整个icop-database吗？
之所以会用到icop-refer，则是因为导出的时候也会模拟序列化操作。首先导出应该只负责接收数据并将其导出到excel，至于数据怎么来的它不需要关心；其次导出通常是导出大量数据，如果在某个时刻缓存失效，那么每一条数据都去查n次数据库，岂不是会非常的慢？
在设计类库的时候要分析库的边界。
* 库内部低内聚
库应该是由相关性很强的代码组成，只负责某一类功能。可是我们看下库icop-pubapp-platform，它里面即包含持久化，又包含导入导出和打印，还做了es数据同步等，这意味着我们要修改其中的任意一项功能，都可能会影响到其它的功能。
* 库依赖冗余
很多库并没有用到其他库，却又引入了它们。最典型的就是库icop-core，它里面定义了大量乱七八糟却没用到的库。
* 三层架构类膨胀
三层架构，即表现层、业务逻辑层和数据访问层。通常只需要1个contronller、2个service（接口+实现）、2个dao（接口加实现），共计5个类，可是我们现在即使做个简单的增删改查业务，都会创建15个类。
* 代码生成
代码生成可以帮我们快速开发，比如上面说的15个类就是通过代码生成来快速创建的，但问题在于如果我们后期要对生成的代码做修改和扩展，还是需要一个个类的去调整。如果是普遍性的实现就应该考虑封装起来，Don't Repeat Yourself ！
* 引用值序列化
引用，即我们所说的“参照”。目前的序列化可以快速将vo中某个外键序列化为包含了id\name\code等属性的json串，但它的问题在于如果调用者基于我们的api文档做开发，他识别不了每个外键到底是返回字符串、JSON对象还是JSON数组，且如果有vo用到了id\code\name之外的冗余字段，那么在序列化后的值里，这些冗余字段的取值很可能是null。

## 扩展
* 对于上面提到的不足之处，我按自己的理解都做了修复
* 引入了swagger2，自动生成后端接口文档（RESTful）
* 审批状态回写支持使用jpa
* 弃审校验增加时间维度校验：引用实体的创建日期大于被引用实体的审批日期，被引用实体不能弃审
* 提供返回值统一处理，不必再在代码各处都手动创建JsonBackData对象，也增加了接口文档的可读性
* 新建导入导出类库，提供更多场景下的支持
* 增加子表非空和子表不能重复两个数据校验注解

## Why Spring Boot
* Spring官方推荐
* Github上星数达到34.4k（Spring Framework星数26.7k）
* 外面公司都在用Spring Boot
* 外面招人都要求会Spring Boot
* 多年以前，Spring Framework成为web框架事实上的标准；现在，Spring Boot就是web框架事实上的标准
* 现在，最火的不是Spring Framework，不是Spring Boot，是Spring cloud，而Spring Cloud基于Spring Boot（Spring Boot基于Spring Framework）

上面是讲大环境，下面看看Spring Boot具体的特性：
* 快速构建Spring应用，很快
* 进可开箱即用，退可按需配置
* 提供指标、健康检查等非功能特性
* 不用生成代码，没有XMl配置

在Spring官网有这样的描述：
![Spring](/images/ijz/Spring.png)
* Spring Boot构建一切
* Spring Cloud协调一切
* Spring Cloud Data Flow 连接一切