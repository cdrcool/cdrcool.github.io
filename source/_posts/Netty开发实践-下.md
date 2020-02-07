---
title: Netty开发实践-下
date: 2020-02-07 21:54:00
categories: Netty
---
在[Netty开发实践-上]()中，我们通过一些示例介绍了如何使用 Netty 进行服务器端/客户端编程，虽然我在示例代码中都做了注释，但是为了更深入的掌握 Netty 编程，接下来我们再做一些知识点梳理，其中有些知识点在前面已经用到了，有些知识点没有用到但仍是我们需要了解的。

### IO 模型切换
我们知道，IO 模型分为：BIO、NIO 和 AIO 三种。在 Netty 中，AIO 相关代码已经移除，BIO 相关代码虽然保留，但是已经不再推荐，也就是说，在以后的 Netty 中，我们可能只能以 NIO 的方式进行编程。
在之前的示例代码中，我们选择的都是 NIO 模型，如下面这样：

**服务器端：**
```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();

ServerBootstrap b = new ServerBootstrap().channel(NioServerSocketChannel.class)...
```

**客户端：**
```java
EventLoopGroup group = new NioEventLoopGroup();

Bootstrap bootstrap = new Bootstrap().channel(NioSocketChannel.class)...
```

那如果我们一定要用 BIO 模型，又该怎么切换呢？跟简单，只需要调整相关类名即可：

**服务器端：**
```java
EventLoopGroup bossGroup = new OioEventLoopGroup(1);
EventLoopGroup workerGroup = new OioEventLoopGroup();

ServerBootstrap serverBootstrap = new ServerBootstrap().channel(OioServerSocketChannel.class)...
```

**客户端：**
```java
EventLoopGroup group = new OioEventLoopGroup();

Bootstrap bootstrap = new Bootstrap().channel(OioSocketChannel.class)...
```

### Reactor 模式切换
在 NIO 模型中，它使用开发模式我们称之为 Reactor 模式。Reactor 模式分为：单线程 Reactor 模式、多线程 Reactor 模式 和 主从 Reactor 模式 三种。
在之前的示例代码中，我们选择的都是 主从 Reactor 模式，如下面这样：

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();

ServerBootstrap serverBootstrap = new ServerBootstrap().group(bossGroup, workerGroup)...
```

那如果我们想要使用单线程 Reactor 模式或 多线程 Reactor 模式，又该怎么切换呢？跟简单，只需要调整相关参数即可：

**使用单线程 Reactor 模式：**
```java
EventLoopGroup eventLoopGroup = new NioEventLoopGroup(1);

ServerBootstrap serverBootstrap = new ServerBootstrap().group(eventLoopGroup)...
```

**使用多线程 Reactor 模式：**
```java
EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

ServerBootstrap serverBootstrap = new ServerBootstrap().group(eventLoopGroup)...
```

## 粘包/半包处理
Netty 对三种常用封帧方式都提供了支持：
方式\支持 | 解码 | 编码
:-: | :-: | :-:
固定长度 | FixedLengthFrameDecoder | 简单
分隔符 | DelimiterBasedFrameDecoder | 简单
固定长度字段存内容的长度信息 | LengthFieldBasedFrameDecoder | LengthFieldPrepender

在之前的示例代码中，我们选择的都是“固定长度字段存内容的长度信息”方式，如下面这样：

```java
/**
 * 一次解码器：处理粘包/半包
 */
public class OrderFrameDecoder extends LengthFieldBasedFrameDecoder {

    public OrderFrameDecoder() {
        super(Integer.MAX_VALUE, 0, 2, 0, 2);
    }
}

/**
 * 一次编码器：处理粘包/半包
 */
public class OrderFrameEncoder extends LengthFieldPrepender {

    public OrderFrameEncoder() {
        super(2);
    }
}
```

由于另外两种都各有弊端，这里就不演示怎么切换了。

