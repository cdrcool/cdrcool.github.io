---
title: Netty IO模型
date: 2020-02-04 22:59:00
categories: Netty
---
## IO 模型
IO 模型主要分类：
* 同步（synchronous）IO 和异步（asynchronous）IO
* 阻塞（blocking）IO 和非阻塞（non-blocking）IO

* 同步阻塞（blocking-IO）简称 BIO
* 同步非阻塞（non-blocking-IO）简称 NIO
* 异步非阻塞（synchronous-non-blocking-IO）简称 AIO

### 同步 VS 异步
同步：发送一个请求，等待返回，再发送下一个请求，同步可以避免出现死锁，脏读的发生。

异步：发送一个请求，不等待返回，随时可以再发送下一个请求，可以提高效率，保证并发。

简述：数据就绪后需要自己去读是同步，数据就绪直接读好再回调给程序是异步。

### 阻塞 VS 非阻塞
阻塞：传统的 IO 流都是阻塞式的。也就是说，当一个线程调用 read（）或者 write（）方法时，该线程将被阻塞，直到有一些数据读读取或者被写入，在此期间，该线程不能执行其他任何任务。在完成网络通信进行 IO 操作时，由于线程会阻塞，所以服务器端必须为每个客户端都提供一个独立的线程进行处理，当服务器端需要处理大量的客户端时，性能急剧下降。

非阻塞：Java NIO 是非阻塞式的。当线程从某通道进行读写数据时，若没有数据可用时，该线程会去执行其他任务。线程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出通道。因此 NIO 可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。

简述：没有数据传过来时，读会阻塞直到有数据；缓冲区满时，写操作也会阻塞，这就是阻塞。非粗赛遇到这些情况，都是直接返回。

### 举例说明
以我们去饭店吃饭为例：
* BIO：食堂排队打饭模式，排队在窗口，打好才走；
* NIO：等待被叫模式，等待被叫，好了自己去端；
* AIO：包厢模式，点单后菜直接被端上桌。

### IO VS NIO
IO | NIO
:-: | :-:
面向流（Stream Oriented） | 面向缓冲区（Buffer Oriented）
阻塞 IO（Blocking IO） | 非阻塞 IO（Non Blocking IO）
无 | 选择器（Selectors）

### BIO VS NIO VS AIO
BIO | NIO | AIO
:-: | :-: | :-:
Thread-Per-Connection | Reactor | Proactor

### BIO、NIO、AIO 适用场景
* BIO 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4 以前的唯一选择。
* NIO 方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂。
* AIO方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用 OS 参与并发操作，编程比较复杂，JDK7 开始支持。

### 多路复用 IO（IO Multiplexing）
即经典的 **Reactor 设计模式**，有时也称为**异步阻塞 IO**，Java 中的 Selector 和 Linux 中的 epoll 都是这种模型。

#### Reactor 模式
Reactor 模式的核心流程：注册感兴趣的事件 -> 扫描是否有感兴趣的事件发生 -> 事件发生后做出响应的处理。

Client/Server | SocketChannel/ServerSocketChannel | OP_ACCEPT | OP_CONNECT | OP_WRITE | OP_READ
:-: | :-: | :-: | :-: | :-: | :-:
client | SocketChannel | | Y | Y | Y
server | ServerSocketChannel | Y | | |
server | SocketChannel | | | Y | Y

Reactor 模式有三种实现方式：Reactor 单线程、Reactor 多线程模式、Reactor 主从模式。

1. Reactor 单线程模式

![Reactor 单线程模式](/images/netty/Reactor单线程模式.png)

每个客户端发起连接请求都会交给 acceptor，acceptor 根据事件类型交给线程 handler 处理，注意 acceptor 处理和 handler 处理都在一个线程中处理，所以其中某个 handler 阻塞时，会导致其他所有的 client 的 handler 都得不到执行，并且更严重的是，handler 的阻塞也会导致整个服务不能接收新的 client 请求(因为 acceptor 也被阻塞了)。 因为有这么多的缺陷，因此单线程 Reactor 模型用的比较少。

2. Reactor 多线程模式

![Reactor 多线程模式](/images/netty/Reactor多线程模式.png)

有专门一个线程，即 Acceptor 线程用于监听客户端的TCP连接请求。

客户端连接的 IO 操作都是由一个特定的 NIO 线程池负责。每个客户端连接都与一个特定的 NIO 线程绑定，因此在这个客户端连接中的所有 IO 操作都是在同一个线程中完成的。

客户端连接有很多，但是 NIO 线程数是比较少的，因此一个 NIO 线程可以同时绑定到多个客户端连接中。

缺点：如果我们的服务器需要同时处理大量的客户端连接请求或我们需要在客户端连接时，进行一些权限的检查，那么单线程的 Acceptor 很有可能就处理不过来，造成了大量的客户端不能连接到服务器。

3. Reactor 主从模式

![Reactor 主从模式](/images/netty/Reactor主从模式.png)

**举例说明：**
以饭店规模变化为例：
* Reactor 单线程模式：一个人包揽所有，迎宾、点菜、做饭、上菜、送客等；
* Reactor 多线程模式：多招几个伙计，大家一起做上面的事情；
* Reactor 主从模式：进一步分工，搞一个或多个人专门做迎宾。

#### 核心概念
1. 缓冲区 Buffer
Buffer 是一个对象。它包含一些要写入或者读出的数据。
在面向流的 I/O 中，可以将数据写入或者将数据直接读到 Stream 对象中。在 NIO 中，所有的数据都是用缓冲区处理。这也就是很多博客说，IO 是面向流的，NIO 是面向缓冲区的。
缓冲区实质是一个数组，通常它是一个字节数组（ByteBuffer），也可以使用其他类的数组。但是一个缓冲区不仅仅是一个数组，缓冲区提供了对数据的结构化访问以及维护读写位置（limit）等信息。
最常用的缓冲区是 ByteBuffer，一个 ByteBuffer 提供了一组功能于操作 byte 数组。除了 ByteBuffer，还有其他的一些缓冲区，事实上，每一种Java基本类型（除了Boolean）都对应一种缓冲区，具体如下：

* ByteBuffer：字节缓冲区
* CharBuffer：字符缓冲区
* ShortBuffer：短整型缓冲区
* IntBuffer：整型缓冲区
* LongBuffer：长整型缓冲区
* FloatBuffer：浮点型缓冲区
* DoubleBuffer：双精度浮点型缓冲区

2. 通道 Channel
Channel 是一个通道，可以通过它读取和写入数据，它就像自来水管一样，网络数据通过 Channel 读取和写入。
通道和流不同之处在于通道是双向的，流只是在一个方向移动，而且通道可以用于读，写或者同时用于读写。因为 Channel 是全双工的，所以它比流更好地映射底层操作系统的 API，特别是在 UNIX 网络编程中，底层操作系统的通道都是全双工的，同时支持读和写。
Channel 有四种实现：

* FileChannel：文件中读取数据。
* DatagramChannel：从 UDP 网络中读取或者写入数据。
* SocketChannel：从 TCP 网络中读取或者写入数据。
* ServerSocketChannel：允许你监听来自 TCP 的连接，就像服务器一样。每一个连接都会有一个 SocketChannel 产生。这句话的意思就是，ServerSocketChannel 用于服务端，SocketChannel 用于客户端。

3. 多路复用器 Selector
Selector 会不断轮询注册在上面的 Channel，如果某个 Channel 上面有新的 TCP 连接接入、读或写事件，这个channel就处于就绪状态，会被 Selector 轮询出来，然后通过 SelectionKey 可以获取 Channel 的集合，进行后续的 I/O 操作。一个多路复用器 Selector 可以同时轮询多个 Channel，由于 JDK 使用了 epoll() 代替传统的 select 实现，所以它没有最大连接句柄 1024/2048 的限制。这就意味着只需一个线程负责 Selector 的轮询，就可以接入成千上万的客户端。

## Netty IO 模型
目前 Netty 只支持 NIO 模型。

## 参考资料
1. [高并发编程系列：NIO、BIO、AIO的区别，及NIO的应用和框架选型](https://youzhixueyuan.com/java-nio-introduce.html)
2. [NIO相关概念介绍：缓冲区Buffer,通道Channel,多路复用器Selector](https://blog.csdn.net/u011521382/article/details/81069457)