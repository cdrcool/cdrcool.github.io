---
title: ijz后端框架(九)引用值序列化&反序列化
date: 2019-02-22 14:41:34
categories: ijz后端框架
tags:
	- jackson
---
当存在实体引用时，我们会在前后端数据传递过程中做对引用字段做序列化与反序列化，也会在审批流弃审时做引用校验，本文主要是谈谈我对引用值序列化&反序列化的理解，弃审校验将在文章[审批扩展](https://cdrcool.github.io/2019/02/22/ijz后端框架%28十%29审批扩展/)中再分析。

## 反序列化
我认为反序列化是可以不要的，因为它就是把前端传递的值（字符串|json对象|json数组）解析成字符串，前端同样可以完成，无非是在表单保存的时候多一行赋值代码（封装到前端组件后，一样不必每处手动赋值）。
现在有点奇怪的是，假如我想保存冗余字段名称或编码，尽管前端已经传递了对象`{"id": "xx", "code": "xx", "name": "xx"}`，但由于反序列化后后端只会接收到id，这样前端还得另外传递名称或编码。
## 序列化
同样，我认为序列化也可以不要，或者可以换一种实现方式。虽然它给开发者提供了便利，但如果服务调用者只是看我们API文档，他并不清楚某个外键字段到底是返回String、JSON Object还是JSON Array，JSON对象里面除了id、code、name外又包含哪些额外的字段。
我认为vo返回哪些字段应该明确在类里面定义，所以如果一定要做序列化，我会采用下面的方式：
```java
@JsonSerialize(using = ReferSerializer.class)
public static class ExampleVO {
    @Refer(billTypes = "company", foreignKey = "id")
    private String companyId;

    @ByRefer(referField = "companyId", fieldName = "name")
    private String companyName;
    @ByRefer(referField = "companyId", fieldName = "code")
    private String companyCode;
    @ByRefer(referField = "companyId", fieldName = "type")
    private String companyType;
    ...
}
```
@Refer注解指定哪个字段是外键，从哪些单据类型对应的实体中去查找，以及外键字段在原实体中的属性名。@ByRefer注解则指定要返回哪些字段，及两个实体间字段的对应关系。
## 缓存使用方式
由于序列化时首先会去缓存中查，缓存中没有再去查数据库，所以目前我们保存数据时会同步保存到redis中。
我认为缓存应该只缓存热点数据，而不是所有的数据都存里面，且我们应该考虑缓存过期策略。
有人会说只是同步缓存了id、code和name，影响不大，且不考虑是否真影响不大，其实我们并不只是同步缓存了id、code和name。比如在上面的ExampleVO类中,除了id、code和name外，我们还用到了companyType。按目前实现方式，要想序列化时得到companyType的值，我们会在注解@ReferSerialTransfer中设置extraFileds属性，这时有3种可能情况：
* 缓存中没有数据，companyType有值
* 缓存中有数据，但只有id、code和name，companyType没值
* 缓存中有数据，但除id、code和name外，只有companyLevel（其他实体写入缓存的），companyType没值

由此可见，除非我们分析每个实体，把其可能被用的字段都写入缓存，否则很可能序列化中某个想要的字段查不到值，但是这样一是麻烦，二是会浪费大量内存。
我的实现里是在查数据之后再把数据写入缓存，并设置一个过期时限。
## 兼容性
在公有云合约系统中，像contractName、contractCode等字段都是在公共表里，但现在更新状态都是直接去更新合同表，所以我在各类合同表里也存了份字段。不过如果升级Hibernate到最新版本，又有新的问题：hibernate不会再自动同步更新公共表和实体表，
所以在改造后的实现中，我会依据属性`ijz.bill.persistenceType`的值（jpa|jdbc）来决定是使用`EntityManager`还是`JdbcTemplate`更新表字段。

