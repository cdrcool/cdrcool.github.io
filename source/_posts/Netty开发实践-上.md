---
title: Netty开发实践-上
date: 2020-02-07 19:18:00
categories: Netty
---
## Maven 依赖
```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
</dependency>
```

## 简单示例
### Echo 示例
我们先用官方示例 echo 来简单了解下如何进行 Netty 开发。该示例类似我们打兵乓球一样，把一个消息从客户端发送到服务器，服务器接收消息后，再把该消息返回给客户端，循环往复。

#### 服务器端代码
```java
/**
 * Echoes back any received data from a client.
 */
public final class EchoServer {

    public static void main(String[] args) throws Exception {
        // Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        final EchoServerHandler serverHandler = new EchoServerHandler();
        try {
            ServerBootstrap b = new ServerBootstrap()
                    // 主从 Reactor 模式
                    .group(bossGroup, workerGroup)
                    // ServerSocketChannel IO 模式
                    .channel(NioServerSocketChannel.class)
                    // 最大等待数量：当服务器请求处理线程全满时，用于临时存放已完成三次握手的请求的队列的最大长度
                    .option(ChannelOption.SO_BACKLOG, 100)
                    // ServerSocketChannel Handler
                    .handler(new LoggingHandler(LogLevel.INFO))
                    // SocketChannel Handler
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) {
                            ChannelPipeline p = ch.pipeline();
                            p.addLast(new LoggingHandler(LogLevel.INFO));
                            p.addLast(serverHandler);
                        }
                    });

            // Start the server.
            ChannelFuture f = b.bind(8007).sync();

            // Wait until the server socket is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

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

#### 客户端代码
```java
/**
 * Sends one message when a connection is open and echoes back any received
 * data to the server.  Simply put, the echo client initiates the ping-pong
 * traffic between the echo client and server by sending the first message to
 * the server.
 */
public final class EchoClient {

    public static void main(String[] args) throws Exception {
        // Configure the client.
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap()
                    .group(group)
                    // SocketChannel IO 模式
                    .channel(NioSocketChannel.class)
                    // 是否启用 Nagle 算法（通过将小的碎片数据连接成更大的报文来提高发送效率）
                    // 如果需要发送一些较小的报文，则需要禁用该算法
                    .option(ChannelOption.TCP_NODELAY, true)
                    // SocketChannel Handler
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) {
                            ChannelPipeline p = ch.pipeline();
                            p.addLast(new LoggingHandler(LogLevel.INFO));
                            p.addLast(new EchoClientHandler());
                        }
                    });

            // Start the client.
            ChannelFuture f = b.connect("127.0.0.1", 8007).sync();

            // Wait until the connection is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down the event loop to terminate all threads.
            group.shutdownGracefully();
        }
    }
}

/**
 * Handler implementation for the echo client.  It initiates the ping-pong
 * traffic between the echo client and server by sending the first message to
 * the server.
 */
public class EchoClientHandler extends ChannelInboundHandlerAdapter {

    private final ByteBuf firstMessage;

    /**
     * Creates a client-side handler.
     */
    public EchoClientHandler() {
        firstMessage = Unpooled.wrappedBuffer("I am echo message".getBytes());
    }

    /**
     * 客户端连接时触发
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        // 连接之后立马发送消息
        ctx.writeAndFlush(firstMessage);
    }

    /**
     * 收到消息时触发
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 返回消息给服务器
        ctx.write(msg);
    }

    /**
     * 消息读取完毕后触发
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws InterruptedException {
        TimeUnit.SECONDS.sleep(3);
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

### Hello Word 示例
完成入门级示例后，我们再来了解下，如何使用 Netty 来实现类似 Spring MVC 的 Web 服务。下面的示例同样来自官方，其实现功能为当我们访问“http://127.0.0.1:8080/”时，服务器会返回“Hello,Word”给浏览器。

#### 服务器端代码
```java
/**
 * An HTTP server that sends back the content of the received HTTP request
 * in a pretty plaintext form.
 */
public final class HttpHelloWorldServer {

    public static void main(String[] args) throws Exception {
        // Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap()
                    // 主从 Reactor 模式
                    .group(bossGroup, workerGroup)
                    // ServerSocketChannel IO 模式
                    .channel(NioServerSocketChannel.class)
                    // 最大等待数量：当服务器请求处理线程全满时，用于临时存放已完成三次握手的请求的队列的最大长度
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    // ServerSocketChannel Handler
                    .handler(new LoggingHandler(LogLevel.INFO))
                    // SocketChannel Handler
                    .childHandler(new HttpHelloWorldServerInitializer());

            // Start the server.
            Channel ch = b.bind(8080).sync().channel();

            System.err.println("Open your web browser and navigate to http://127.0.0.1:8080/");

            // Wait until the server socket is closed.
            ch.closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

/**
 * 在 Channel 注册到 EventLoop 后，对这个 Channel 执行一些初始化操作。
 * ChannelInitializer 虽然会在一开始会被注册到 Channel 相关的 pipeline 里，但是在初始化完成之后，ChannelInitializer 会将自己从 pipeline 中移除，
 * <p>
 * 使用场景：
 * 1. 在 ServerBootstrap 初始化时，为监听端口 accept 事件的 Channel 添加 ServerBootstrapAcceptor
 * 2. 在有新链接进入时，为监听客户端 read/write 事件的 Channel 添加用户自定义的 ChannelHandler
 */
public class HttpHelloWorldServerInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    public void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();
        p.addLast(new HttpServerCodec());
        p.addLast(new HttpServerExpectContinueHandler());
        p.addLast(new HttpHelloWorldServerHandler());
    }
}

/**
 * 业务处理：接收 Http 请求，返回 Hello,Word
 * <p>
 * 继承自{@link SimpleChannelInboundHandler}，好处在于：
 * 1. 能自动释放{@link io.netty.buffer.ByteBuf}
 * 2. 增加消息类型匹配，当消息类型不匹配时什么都不做
 */
public class HttpHelloWorldServerHandler extends SimpleChannelInboundHandler<HttpObject> {
    private static final byte[] CONTENT = {'H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd'};

    /**
     * 消息读取完毕后触发
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    /**
     * 收到消息时触发
     */
    @Override
    public void channelRead0(ChannelHandlerContext ctx, HttpObject msg) {
        // 只处理 Http 请求
        if (msg instanceof HttpRequest) {
            HttpRequest req = (HttpRequest) msg;

            boolean keepAlive = HttpUtil.isKeepAlive(req);
            FullHttpResponse response = new DefaultFullHttpResponse(req.protocolVersion(), OK,
                    Unpooled.wrappedBuffer(CONTENT));
            response.headers()
                    .set(CONTENT_TYPE, TEXT_PLAIN)
                    .setInt(CONTENT_LENGTH, response.content().readableBytes());

            if (keepAlive) {
                if (!req.protocolVersion().isKeepAliveDefault()) {
                    response.headers().set(CONNECTION, KEEP_ALIVE);
                }
            } else {
                // Tell the client we're going to close the connection.
                response.headers().set(CONNECTION, CLOSE);
            }

            ChannelFuture f = ctx.write(response);

            if (!keepAlive) {
                f.addListener(ChannelFutureListener.CLOSE);
            }
        }
    }

    /**
     * 发生异常时触发
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

可能大家注意到了，这里没有对客户端编程，确实，因为这里的客户端就是我们的浏览器。

## 网络应用程序编程基本步骤
上面两个示例都非常简单，接下来我们会模拟真实订单场景（当然还是很简单的），以此来了解 Netty 更多的特性。但是在具体示例之前，我们有必要先了解下网络应用程序编程基本步骤。

![网络应用程序编程基本步骤1](/images/netty/网络应用程序编程基本步骤1.png)

![网络应用程序编程基本步骤2](/images/netty/网络应用程序编程基本步骤2.png)

## 点单示例
### Maven 依赖
由于用到了 Json 编解码，和 guava 中的一些工具类，因此除了 Netty 依赖外，还需要添加以下依赖：
```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
</dependency>
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
</dependency>
```

### 案例介绍
在本示例中，我们会模拟饭店点单：在哪一桌，要吃什么菜。因为示例本身是为了让我们了解 Netty 而非业务，所以本示例的业务非常简单，就是发送客户端一个订单操作给服务器，服务器收到请求后，再返回订单结果给客户端。

![点单案例介绍](/images/netty/点单案例介绍.png)

如上图，我们处理 OrderOperation 之外，还定义了 AuthOperation 和 KeepOperation，这是为了演示 Netty 的高级特性。

### 数据结构设计
**消息对象数据结构：**
![消息对象数据结构](/images/netty/消息对象数据结构.png)

1. 消息体由消息头和消息体组成
2. 消息头由版本号（version）、操作码（opCode）和消息id（streamId）组成
3. 消息体使用 Json 编解码，既可以是操作对象（operation），也可以是操作结果对象（operation result）
4. 传输协议使用 TCP，因此要解决粘包/半包问题，因此消息对象需要指定 length

**UML：**
![点单示例UML](/images/netty/点单示例UML.png)

1. 消息对象基类 BaseMessage 有两个属性：消息头对象 MessageHeader 和 消息体对象 BaseMessageBody 基类，消息体对象具体类型由泛型 T 指定
2. 操作对象基类 BaseOperation 和操作结果对象基类 BaseOperationResult 都继承自消息体对象 BaseMessageBody 基类
3. 操作对象基类 BaseOperation 有 3 个子类：点单操作对象 OrderOperation、认证操纵对象 OrderOperation 和心跳检测操作对象 KeepaliveOperation
4. 操作结果对象 BaseOperationResult 基类有 3 个子类：点单操作结果对象 OrderOperationResult、认证操纵结果对象 OrderOperationResult 和心跳检测操作结果对象 KeepaliveOperationResult，分别对应各自的操作对象
5. 消息对象基类 BaseMessage 有 2 个子类：请求对象 RequestMessage 和响应对象 ResponseMessage。

### 服务器端编程
#### 编解码
* 一次解码器 OrderFrameDecoder：处理粘包/半包
```java
/**
 * 一次解码器：处理粘包/半包
 */
public class OrderFrameDecoder extends LengthFieldBasedFrameDecoder {

    public OrderFrameDecoder() {
        super(Integer.MAX_VALUE, 0, 2, 0, 2);
    }
}
```

* 一次编码器 OrderFrameEncoder：处理粘包/半包
```java
/**
 * 一次编码码器：处理粘包/半包
 */
public class OrderFrameEncoder extends LengthFieldPrepender {

    public OrderFrameEncoder() {
        super(2);
    }
}
```

* 二次解码器 OrderProtocolDecoder：将 ByteBuf 解析为 RequestMessage
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
```

* 二次编码器 OrderProtocolEncoder：将 ResponseMessage 解析为 ByteBuf
```java
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
```

#### 业务处理
```java
/**
 * 业务处理 Handler
 * <p>
 * 继承自{@link SimpleChannelInboundHandler}，好处在于：
 * 1. 能自动释放{@link io.netty.buffer.ByteBuf}
 * 2. 增加消息类型匹配，当消息类型不匹配时什么都不做
 */
public class OrderServerProcessHandler extends SimpleChannelInboundHandler<RequestMessage> {

    /**
     * 读取并处理数据
     * 接收{@link RequestMessage}类型数据，如果数据类型不为{@link RequestMessage}，则什么都不做
     */
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RequestMessage msg) {
        BaseOperation operation = msg.getBody();
        BaseOperationResult operationResult = operation.execute();

        ResponseMessage responseMessage = new ResponseMessage();
        responseMessage.setHeader(msg.getHeader());
        responseMessage.setBody(operationResult);

        ctx.writeAndFlush(responseMessage);
    }
}
```

#### 读监测
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

#### 服务器 Server
```java
/**
 * 服务器
 */
public class Server {

    public static void main(String[] args) throws InterruptedException {
        // 即使这里设置了多线程，实际还是只会用到一个线程
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap serverBootstrap = new ServerBootstrap()
                    // 设置 ServerSocketChannel IO 模式
                    .channel(NioServerSocketChannel.class)
                    // 设置主从 Reactor 模式
                    .group(bossGroup, workerGroup)
                    // 增加日志输出
                    .handler(new LoggingHandler(LogLevel.INFO))
                    // SocketChannel Handler
                    .childHandler(new ChannelInitializer<NioSocketChannel>() {
                        @Override
                        protected void initChannel(NioSocketChannel ch) {
                            ChannelPipeline pipeline = ch.pipeline();

                            // 启用读空闲监测：及时清理空闲的连接
                            pipeline.addLast(new ServerIdleCheckHandler());

                            // 特别需要注意编解码顺序
                            // 先接收后发送 -> 先解码（一次解码 -> 二次解码）后编码（二次编码 -> 一次编码）
                            pipeline.addLast(new OrderFrameDecoder());
                            pipeline.addLast(new OrderFrameEncoder());
                            pipeline.addLast(new OrderProtocolEncoder());
                            pipeline.addLast(new OrderProtocolDecoder());

                            // 增加日志输出
                            // 不同位置输出的内容不同
                            pipeline.addLast(new LoggingHandler(LogLevel.INFO));

                            // 业务处理 Handler 放到最后添加
                            pipeline.addLast(new OrderServerProcessHandler());

                        }
                    });
            // 启动服务器，且保持阻塞
            ChannelFuture channelFuture = serverBootstrap.bind(8090).sync();

            // 等待直到 Server Socket 关闭
            channelFuture.channel().closeFuture().sync();
        } finally {
            // 注意先后顺序：先关闭 bossGroup（不再接收新请求），后关闭 workerGroup
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

### 客户端编程
#### 编解码
* 一次解码器 OrderFrameDecoder：处理粘包/半包
```java
/**
 * 一次解码器：处理粘包/半包
 */
public class OrderFrameDecoder extends LengthFieldBasedFrameDecoder {

    public OrderFrameDecoder() {
        super(Integer.MAX_VALUE, 0, 2, 0, 2);
    }
}
```

* 一次编码器 OrderFrameEncoder：处理粘包/半包
```java
/**
 * 一次编码码器：处理粘包/半包
 */
public class OrderFrameEncoder extends LengthFieldPrepender {

    public OrderFrameEncoder() {
        super(2);
    }
}
```

* 二次解码器 OrderProtocolDecoder：将 ByteBuf 解析为 ResponseMessage
```java
/**
 * 二次解码器：将 {@link ByteBuf} 解析为 {@link ResponseMessage}
 */
public class OrderProtocolDecoder extends MessageToMessageDecoder<ByteBuf> {

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) {
        ResponseMessage responseMessage = new ResponseMessage();
        responseMessage.decode(msg);

        out.add(responseMessage);
    }
}
```

* 二次编码器 OrderProtocolEncoder：将 RequestMessage 解析为 ByteBuf
```java
/**
 * 二次码码器：将 {@link RequestMessage} 解析为 {@link ByteBuf}
 */
public class OrderProtocolEncoder extends MessageToMessageEncoder<RequestMessage> {

    @Override
    protected void encode(ChannelHandlerContext ctx, RequestMessage msg, List<Object> out) {
        ByteBuf byteBuf = ctx.alloc().buffer();
        msg.encode(byteBuf);

        out.add(byteBuf);
    }
}
```

* 二次编码器 OperationToRequestMessageEncoder，将 BaseOperation 转换为 RequestMessage，以实现自动添加消息 id
```java
/**
 * 二次编码器：将 {@link BaseOperation} 转换为 {@link RequestMessage}，以实现自动添加消息 id
 */
public class OperationToRequestMessageEncoder extends MessageToMessageEncoder<BaseOperation> {

    @Override
    protected void encode(ChannelHandlerContext ctx, BaseOperation msg, List<Object> out) {
        RequestMessage requestMessage = new RequestMessage(IdUtil.nextId(), msg);
        out.add(requestMessage);
    }
}
```

#### 业务处理
```java
/**
 * 响应分发 Handler
 * <p>
 * 继承自{@link SimpleChannelInboundHandler}，好处在于：
 * 1. 能自动释放{@link io.netty.buffer.ByteBuf}
 * 2. 增加消息类型匹配，当消息类型不匹配时什么都不做
 */
@AllArgsConstructor
public class ResponseDispatcherHandler extends SimpleChannelInboundHandler<ResponseMessage> {
    private RequestPendingCenter requestPendingCenter;

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ResponseMessage msg) {
        // 将响应分发到对应的 streamId
        requestPendingCenter.set(msg.getHeader().getStreamId(), msg.getBody());
    }
}

public class RequestPendingCenter {
    /**
     * 记录 streamId 与 OperationResultFuture 的对应关系
     */
    private Map<Long, OperationResultFuture> map = new ConcurrentHashMap<>();

    /**
     * 添加 entry
     */
    public void add(long streamId, OperationResultFuture future) {
        this.map.put(streamId, future);
    }

    /**
     * 设置 future 对象的值，并从 map 中移除 streamId
     */
    public void set(long streamId, BaseOperationResult result) {
        OperationResultFuture future = this.map.get(streamId);
        if (future != null) {
            future.setSuccess(result);
            this.map.remove(streamId);
        }
    }
}

/**
 * Future 对象，其结果为 BaseOperationResult
 */
public class OperationResultFuture extends DefaultPromise<BaseOperationResult> {

}
```

#### 写监测
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

#### 客户端 Client
```java
/**
 * 客户端
 */
@Slf4j
public class Client {

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        EventLoopGroup group = new NioEventLoopGroup();

        RequestPendingCenter requestPendingCenter = new RequestPendingCenter();

        try {
            Bootstrap bootstrap = new Bootstrap()
                    // 设置 SocketChannel IO 模式
                    .channel(NioSocketChannel.class)
                    // 设置 EventLoopGroup
                    .group(group)
                    // SocketChannel Handler
                    .handler(new ChannelInitializer<NioSocketChannel>() {
                        @Override
                        protected void initChannel(NioSocketChannel ch) {
                            ChannelPipeline pipeline = ch.pipeline();

                            // 启用写空闲监测
                            pipeline.addLast(new ClientIdleCheckHandler());

                            // 特别需要注意编解码顺序
                            // 先接收后发送 -> 先解码（一次解码 -> 二次解码）后编码（二次编码 -> 一次编码）
                            pipeline.addLast(new OrderFrameDecoder());
                            pipeline.addLast(new OrderFrameEncoder());
                            pipeline.addLast(new OrderProtocolEncoder());
                            pipeline.addLast(new OrderProtocolDecoder());

                            // 增加日志输出
                            // 不同位置输出的内容不同
                            pipeline.addLast(new LoggingHandler(LogLevel.INFO));

                            // 将响应结果记录到 streamId 对应的 future 中
                            pipeline.addLast(new ResponseDispatcherHandler(requestPendingCenter));

                            // 将响应结果记录到 streamId 对应的 future 中
                            pipeline.addLast(new ResponseDispatcherHandler(requestPendingCenter));

                        }
                    });
            // 启动客户端，且保持阻塞
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 8090).sync();

            // 发送消息
            long streamId = IdUtil.nextId();
            RequestMessage requestMessage = new RequestMessage(streamId, new OrderOperation(1001, "土豆"));
            OperationResultFuture future = new OperationResultFuture();
            // 记录 streamId 与 future 的对应关系
            requestPendingCenter.add(streamId, future);

            channelFuture.channel().writeAndFlush(requestMessage);

            // 阻塞，等待获取结果
            BaseOperationResult result = future.get();
            log.info("Result: {}", result);

            // 等待直到 Server Socket 关闭
            channelFuture.channel().closeFuture().sync();
        } finally {
            // Shut down the event loop to terminate all threads.
            group.shutdownGracefully();
        }
    }
}
```