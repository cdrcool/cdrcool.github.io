---
title: Netty 简介
date: 2020-02-04 20:07:00
categories: Netty
---
Netty 是一个广受欢迎的**异步的、事件驱动的、基于NIO实现的** Java 开源网络应用程序框架，用于快速开发**可维护的高性能协议服务器和客户端**。

## JDK 原生 NIO 程序的问题
JDK 原生也有一套网络应用程序 API，但是存在一系列问题，主要如下：

* NIO 的类库和 API 繁杂，使用麻烦：你需要熟练掌握 Selector、ServerSocketChannel、SocketChannel、ByteBuffer 等。

* 需要具备其他的额外技能做铺垫：例如熟悉 Java 多线程编程，因为 NIO 编程涉及到 Reactor 模式，你必须对多线程和网路编程非常熟悉，才能编写出高质量的 NIO 程序。

* 可靠性能力补齐，开发工作量和难度都非常大：例如客户端面临断连重连、网络闪断、半包读写、失败缓存、网络拥塞和异常码流的处理等等。NIO 编程的特点是功能开发相对容易，但是可靠性能力补齐工作量和难度都非常大。

* JDK NIO 的 Bug：例如臭名昭著的 Epoll Bug，它会导致 Selector 空轮询，最终导致 CPU 100%。官方声称在 JDK 1.6 版本的 update 18 修复了该问题，但是直到 JDK 1.7 版本该问题仍旧存在，只不过该 Bug 发生概率降低了一些而已，它并没有被根本解决。

## Netty 框架的优势
* API使用简单，开发门槛低；
* 功能强大，预置了多种编解码功能，支持多种主流协议；
* 定制能力强，可以通过 ChannelHandler 对通信框架进行灵活地扩展；
* 性能高，通过与其他业界主流的 NIO 框架对比，Netty 的综合性能最优；
* 成熟、稳定，Netty 修复了已经发现的所有 JDK NIO BUG，业务开发人员不需要再为 NIO 的 BUG 而烦恼；
* 社区活跃，版本迭代周期短，发现的BUG可以被及时修复，同时，更多的新功能会加入；
* 经历了大规模的商业应用考验，质量得到验证。在互联网、大数据、网络游戏、企业应用、电信软件等众多行业得到成功商用，证明了它已经完全能够满足不同行业的商业应用。

## Netty 的应用场景
* 互联网行业：在分布式系统中，各个节点之间需要远程服务调用，高性能的 RPC 框架必不可少，Netty 作为异步高性能的通信框架，往往作为基础通信组件被这些 RPC 框架使用。典型的应用有：阿里分布式服务框架 Dubbo 的 RPC 框架使用 Dubbo 协议进行节点间通信，Dubbo 协议默认使用 Netty 作为基础通信组件，用于实现各进程节点之间的内部通信。

* 游戏行业：无论是手游服务端还是大型的网络游戏，Java 语言得到了越来越广泛的应用。Netty 作为高性能的基础通信组件，它本身提供了 TCP/UDP 和 HTTP 协议栈。非常方便定制和开发私有协议栈，账号登录服务器，地图服务器之间可以方便的通过 Netty 进行高性能的通信。

* 大数据领域：经典的 Hadoop 的高性能通信和序列化组件 Avro 的 RPC 框架，默认采用 Netty 进行跨界点通信，它的 Netty Service 基于 Netty 框架二次封装实现。

* 企业软件：企业和 IT 集成需要 ESB，Netty 对多协议支持、私有协议定制的简洁性和高性能是 ESB RPC 框架的首选通信组件。事实上，很多企业总线厂商会选择 Netty 作为基础通信组件，用于企业的 IT 集成。

* 通信行业：Netty 的异步高性能、高可靠性和高成熟度的优点，使它在通信行业得到了大量的应用。

## Netty 在开源框架中的应用
* 数据库：Cassandra

* 大数据处理：Spark、Hadoop

* 消息队列：RocketMQ

* 搜索引擎：Elasticsearch

* 框架：gRPC、Apache Dubbo、Spring5(WebFlux)

* 分布式协调器：ZooKeeper

* 工具类：async-http-client

* 其它参考：https://netty.io/wiki/adopters.html

## 参考资料
1. [高并发架构系列：Netty的实现原理、特点与优势、以及适用场景](https://youzhixueyuan.com/netty-implementation-principle.html)