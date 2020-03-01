---
title: Netty心跳机制
date: 2020-02-08 14:52:00
categories: Netty
tags:
    - Netty
---
## 何为心跳
顾名思义，所谓心跳，即在 TCP 长连接中，客户端和服务器之间定期发送的一种特殊的数据包，通知对方自己还在线，以确保 TCP 连接的有效性。

## 为什么需要心跳
因为网络的不可靠性，有可能在 TCP 保持长连接的过程中，由于某些突发情况，例如网线被拔出，突然掉电等，会造成服务器和客户端的连接中断。
在这些突发情况下，如果恰好服务器和客户端之间没有交互的话，那么它们是不能在短时间内发现对方已经掉线的。
为了解决这个问题，我们就需要引入心跳机制。
心跳机制的工作原理是：在服务器和客户端之间一定时间内没有数据交互时，即处于 idle 状态时，客户端或服务器会发送一个特殊的数据包给对方，当接收方收到这个数据报文之后，也立即发送一个特殊的数据报文，回应发送方，此即一个 PING-PONMG 交互。自然地，当某一端收到心跳消息后，就知道了对方仍然在线，这就确保 TCP 连接的有效性。

## 如何实现心跳
我们可以通过两种方式实现心跳机制：
* 使用 TCP 协议层里面的 keepalive 机制
* 在应用层上实现自定义的心跳机制（在 Netty 中即为 Idle check）

虽然在 TCP 协议层上，提供了 keepalive 保活机制，但是使用它有几个缺点：
1. 它不是 TCP 的标准协议，并且默认时关闭的。
2. TCP keepalive 机制依赖于操作系统的实现，默认的 keepalive 心跳时间是**两个小时**，并且对 keepalive 的修改需要系统调用（或者修改系统配置），灵活性不够。
3. TCP keepalive 与 TCP 协议绑定，因此如果需要更换为 UDP 协议时，keepalive 机制就失效了。

虽然使用 TCP 层面的 keepalive 机制比自定义的应用层心跳机制节省流量, 但是基于上面的几点缺点, 一般的实践中, 人们大多数都是选择在应用层上实现自定义的心跳。

### TCP keepalive
TCP keepalive 核心参数：
* net.ipv4.tcp_keepalive_time = 7200
* net.ipv4.tcp_keepalive_intl = 75
* net.ipv4.tcp_keepalive_probes = 9
当启用（默认关闭）keepalive 时，TCP 在连接没有数据通过的 7200 秒后发送 keepalive 消息，当探测没有回复时按 75 秒的重试频率重发，一直发 9 个探测包都没有确认时连接失败，所以总耗时一般为：2 小时 11 分（7200 秒 + 75 * 9）。

### Idle check
Idle 监测，只是负责诊断，诊断后，做出不同的行为，决定 Idle 监测的最终用途。
* 发送 keepalive（区别于 TCP keepalive）：在有其它数据传输的时候，不发送 keepalive，无数据传输超过一定时间，判定为 Idle，再发 keepalive。
* 直接关闭连接：快速释放损坏的、恶意的、很久不用的连接，让系统时刻保持最好的状态；简单粗暴，客户端可能需要重连。
实际应用中：结合起来使用。按需 keepalive，保证不会空闲，如果空闲，关闭连接。

## Netty 中开启心跳监测
### Server 端开启 TCP keepalive
* bootstrap.childOption(ChannelOption.SO_KEEPALIVE, true)
* bootstrap.childOption(NioChannelOption.SO_KEEPALIVE, true)

注意，是 **.childOption**(ChannelOption.SO_KEEPALIVE, true) 而不是 .option(ChannelOption.SO_KEEPALIVE, true)，虽然可以设置后者，但是无效。

### 开启 Idle Check
要开启 Idle Check，关键是继承 **IdleStateHandler**。实例化一个 IdleStateHandler 需要提供 3 个参数：
* readerIdleTimeSeconds：读超时. 即当在指定的时间间隔内没有从 Channel 读取到数据时, 会触发一个 READER_IDLE 的 IdleStateEvent 事件。
* writerIdleTimeSeconds：写超时. 即当在指定的时间间隔内没有数据写入到 Channel 时, 会触发一个 WRITER_IDLE 的 IdleStateEvent 事件。
* allIdleTimeSeconds：读/写超时. 即当在指定的时间间隔内没有读或写操作时, 会触发一个 ALL_IDLE 的 IdleStateEvent 事件。

## 参考资料
1. [浅析 Netty 实现心跳机制与断线重连](https://segmentfault.com/a/1190000006931568)