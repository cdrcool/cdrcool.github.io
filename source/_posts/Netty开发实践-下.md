---
title: Netty开发实践-下
date: 2020-02-07 21:54:00
categories: Netty
---
在[Netty开发实践-上](https://cdrcool.github.io/2020/02/07/Netty%E5%BC%80%E5%8F%91%E5%AE%9E%E8%B7%B5-%E4%B8%8A/)中，我们通过一些示例介绍了如何使用 Netty 进行服务器端/客户端编程，虽然我在示例代码中都做了注释，但是为了更深入的掌握 Netty 编程，接下来我们再做一些知识点梳理，其中有些知识点在前面已经用到了，有些知识点没有用到但仍是我们需要了解的。

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
为了处理 TCP 的粘包/半包问题，Netty 提供了三种常用的封帧方式：
方式\支持 | 解码 | 编码
:-: | :-: | :-:
固定长度 | FixedLengthFrameDecoder | 简单
分隔符 | DelimiterBasedFrameDecoder | 简单
固定长度字段存内容的长度信息 | LengthFieldBasedFrameDecoder | LengthFieldPrepender

上面三种解码器都继承自 ByteToMessageDecoder。

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

由于另外两种都不常用，这里就不演示怎么切换了。

## 序列化/反序列化
Netty 对一些常用的序列化/反序列化方式都提供了支持，如 base64、bytes、compression、json、marshaling、protobuf、serialization、string、xml 等，编码器都继承自 MessageToMessageEncoder<I>，解码器都继承自 MessageToMessageDecoder<I>。

在之前的示例代码中，我们选择的都是 json 方式，如下面这样：

```java
/**
 * 二次解码器：将 {@link ByteBuf} 解析为 {@link RequestMessage}
 */
public class OrderProtocolDecoder extends MessageToMessageDecoder<ByteBuf> {

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) {
        RequestMessage requestMessage = new RequestMessage();
        requestMessage.decode(msg);

        out.add(requestMessage);
    }
}

/**
 * 二次编码器：将 {@link ResponseMessage} 解析为 {@link ByteBuf}
 */
public class OrderProtocolEncoder extends MessageToMessageEncoder<ResponseMessage> {

    @Override
    protected void encode(ChannelHandlerContext ctx, ResponseMessage msg, List<Object> out) {
        ByteBuf byteBuf = ctx.alloc().buffer();
        msg.encode(byteBuf);

        out.add(byteBuf);
    }
}

/**
 * 消息对象基类
 */
@Data
public abstract class BaseMessage<T extends BaseMessageBody> {
    private MessageHeader header;
    private T body;

    /**
     * 编码消息
     *
     * @param byteBuf 存储编码后的消息
     */
    public void encode(ByteBuf byteBuf) {
        byteBuf.writeInt(header.getVersion());
        byteBuf.writeLong(header.getStreamId());
        byteBuf.writeInt(header.getOpcode());
        byteBuf.writeBytes(JsonUtil.toJson(body).getBytes());
    }

    /**
     * 根据 opcode 获得对应的 MessageBody
     *
     * @param opcode 操作码
     * @return 消息体
     */
    public abstract Class<T> getMessageBodyDecodeClass(int opcode);

    /**
     * 解码
     *
     * @param msg 解码前的消息
     */
    public void decode(ByteBuf msg) {
        int version = msg.readInt();
        long streamId = msg.readLong();
        int opcode = msg.readInt();

        this.header = MessageHeader.builder()
                .version(version)
                .streamId(streamId)
                .opcode(opcode)
                .build();

        this.body = JsonUtil.fromJson(msg.toString(StandardCharsets.UTF_8), getMessageBodyDecodeClass(opcode));
    }
}
```

在其它官方示例中，演示了如果使用其它方式的编解码，如在 “Word Clock”示例中，演示了如何使用 protobuf，大家可以自行查看相关代码。

## 空闲监测
在应用程序运行中，为了避免一些不怀好意或者无所事事的客户端一直和我们的应用程序保持连接，从而给我们造成资源浪费，因此一般我们都会进行空闲监测。
在 Netty 中，实现空闲监测的核心类是 IdleStateHandler。在前面的订单示例中，我们已经用到了这个类，现在再来回顾下是如何使用它的。

**服务端读监测：**
```java
/**
 * Read Idle Handler
 * 服务器 10s 接收不到 channel 的请求就断掉连接：保护自己、瘦身（及时清理空闲的连接）
 */
@Slf4j
public class ServerIdleCheckHandler extends IdleStateHandler {

    public ServerIdleCheckHandler() {
        super(10, 0, 0, TimeUnit.SECONDS);
    }

    @Override
    protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
        // 如果是第一次 read idle，就断掉连接
        if (evt == IdleStateEvent.FIRST_READER_IDLE_STATE_EVENT) {
            log.info("Idle check happen, so close the connection");
            ctx.close();

            // 因此做了自定义处理，就不再触发该事件
            return;
        }

        // 如果是其它事件，保持触发
        super.channelIdle(ctx, evt);
    }
}
```

**客户端写监测：**
```java
/**
 * Write Idle Handler
 * 客户端 5s 不发送数据就触发一个写空闲事件
 * 配合{@link KeepaliveHandler}使用，以避免连接被断，同时启用不频繁的 keepalive
 */
public class ClientIdleCheckHandler extends IdleStateHandler {

    public ClientIdleCheckHandler() {
        super(0, 5, 0, TimeUnit.SECONDS);
    }
}
```

**客户端写超时，发送 keepalive：**
```java
/**
 * 捕捉写空闲事件：发一个 keepalive
 * 配合{@link ClientIdleCheckHandler}使用，以避免连接被断，同时启用不频繁的 keepalive
 *
 * 因为不涉及内存共享，所以设置为可共享的{@link io.netty.channel.ChannelHandler.Sharable}
 */
@Slf4j
@ChannelHandler.Sharable
public class KeepaliveHandler extends ChannelDuplexHandler {

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        // 捕捉第一次写空闲事件，发送 keepalive
        if (evt == IdleStateEvent.FIRST_WRITER_IDLE_STATE_EVENT) {
            log.info("Write Idle happen, so need to send keepalive to keep connection not closed by server");
            KeepaliveOperation operation = new KeepaliveOperation();
            RequestMessage message = new RequestMessage(IdUtil.nextId(), operation);
            ctx.writeAndFlush(message);
        }
        super.userEventTriggered(ctx, evt);
    }
}
```

## 调优参数
### System 参数
* ulimit
Linux Option。进行 TCP 连接时，系统为每个 TCP 连接创建一个 socket 句柄，也就是一个文件句柄，但是 Linux 对每个进程打开的文件句柄数量做了限制，如果超出，报错“Too many open file”。
注意：ulimit 命令修改的数值**只对当前登录用户**的目前使用环境有效，系统重启或者用户退出后就会失效，所以可以作为程序启动脚本的一部分，让它在程序启动前执行。

* TCP_NODELAY
SocketChannel 参数。设置是否启用 Nagle 算法：通过将小的碎片数据连接成更大的报文来提高发送效率。如果需要发送一些较小的报文，则需要禁用该算法。默认不开启。 

* SO_BACKLOG
ServerSocketChannel 参数。最大的等待连接数量。当服务器请求处理线程全满时，用于临时存放已完成三次握手的请求的队列的最大长度。默认值 128。

* SO_REUSEADDR
SocketChannel/ServerSocketChannel 参数。地址重用，解决“Address already in use”。常用开启场景：多网卡（IP）绑定相同端口；让关闭连接释放的端口更早可使用。默认不开启。

### Netty 参数
* CONNECT_TIMEOUT_MILLIS
SocketChannel 参数。客户端连接服务器最大允许时间。默认值 30s。

* io.netty.eventLoopThreads
System 参数。IO Thread 数量。默认值 availableProcessors * 2。

* io.netty.availableProcessors
System 参数。指定 availableProcessors。考虑 Docker/VM 等情况。

* io.netty.leakDetection.level
System 参数。内存泄漏检测级别：DISABLED/SIMPLE 等。默认值 SIMPLE。

## 跟踪诊断
### 设置线程名
```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("boss"));
EventLoopGroup workerGroup = new NioEventLoopGroup(0, new DefaultThreadFactory("worker"));
```

### 设置 Handler 名
```java
pipeline.addLast("frameDecoder", new OrderFrameDecoder());
```

### 使用日志
```java
// 不同位置输出的内容不同
pipeline.addLast(new LoggingHandler(LogLevel.INFO));
```

## 优化使用
### 整改线程模型
对于 IO 密集型应用，我们通常需要整改线程模型：独立出“线程池”来处理业务（处理业务的线程不再与 NioEventLoop 共享）。
```java
// 对于 IO 密集型应用，独立出“线程池”来处理业务（这里不能使用 NioEventLoopGroup，不然只会使用到 1 个线程）
EventExecutorGroup eventExecutors = new UnorderedThreadPoolEventExecutor(1, new DefaultThreadFactory("business"));

// 业务处理 Handler 放到最后添加
pipeline.addLast(eventExecutors, new OrderServerProcessHandler());
```

### 增强写，延迟与吞吐量的抉择
在前面的点单示例中，我们调用的是`ctx.writeAndFlush(msg)`：写数据之后立刻发送出去，这样虽然延迟降低了，但是吞吐量又会受影响下降。如果我们是要吞吐量优先，那么有下面两种改进方式。

* 利用 channelReadComplete
就像我们在 echo 示例中演示的这样：

```java
/**
 * Handler implementation for the echo server.
 */
@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {

    /**
     * 收到消息时触发
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 返回消息给客户端
        ctx.write(msg);
    }

    /**
     * 消息读取完毕后触发
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    /**
     * 发生异常时触发
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}
```

不过使用 channelReadComplete 也有弊端：
1. 不适合异步业务线程（不复用 Nio event loop）处理
channelRead 中的业务处理结果的 write 很可能发生在 channelReadComplete 之后。

2. 不适合更精细的控制
例如连续读 16 次时，第 16 次是 flush，但是如果保持连续的次数不变，如何做到 3 次就 flush ？

* 使用 flushConsolidationHandler
```java
// 读 5 次之后才 flush，并开启异步增强
pipeline.addLast(new FlushConsolidationHandler(5, true));

// 业务处理 Handler 放到最后添加
pipeline.addLast(eventExecutorGroup, new OrderServerProcessHandler());
```

### 流量整形
```java
// 全局流量整形
GlobalTrafficShapingHandler globalTrafficShapingHandler = new GlobalTrafficShapingHandler(
        new NioEventLoopGroup(), 100 * 1024 * 1024, 100 * 1024 * 1024);

// 启用流量整形
// 只会处理 ByteBuf，因此要注意放置的位置
pipeline.addLast(globalTrafficShapingHandler);
```
