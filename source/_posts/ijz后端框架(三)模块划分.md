---
title: ijz后端框架(三)模块划分
date: 2019-02-22 17:31:08
categories: ijz后端框架
---
结合后端框架现有功能，我重新划分了以下几个模块：
![模块划分结构图](/images/ijz/模块划分结构图.png)
* [ijz-boot-dependencies](https://git.yonyouccs.com/ijz-modules-jars/ijz-boot-dependencies)
基于spring-boot-dependencies，统一定义各类库版本号
* ijz framework
包含[ijz-export](https://git.yonyouccs.com/ijz-modules-jars/ijz-export)和ijz-print等类库（目前只有ijz-export）
* ijz boot starters
    提供各类开箱即用的类库，底层可能会依赖于ijz framework里的类库
    + [bill-spring-boot-starter](https://git.yonyouccs.com/ijz-modules-jars/bill-spring-boot-starter)
        封装单据的持久化、返回值/异常统一处理、数据校验、参照值序列化/反序列化化、审批扩展等基本核心功能
    + [shiro-spring-boot-starter](https://git.yonyouccs.com/ijz-modules-jars/shiro-spring-boot-starter)
    + [dubbox-spring-boot-starter](https://git.yonyouccs.com/ijz-modules-jars/dubbox-spring-boot-starter)
* [ijz-boot-starter-parent](https://git.yonyouccs.com/ijz-modules-jars/ijz-boot-starter-parent)
wen项目需要继承的项目，即具备icop相关能力，又引入了版本号定义。


