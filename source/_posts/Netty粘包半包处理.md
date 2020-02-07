---
title: Netty 粘贴/半包处理
date: 2020-02-07 22:20:00
categories: Netty
---
## 什么是粘包/半包
比如客户端向服务器端发送了两条消息，分别是“ABC”、“DEF”，由于 TCP 是流式协议，服务器端可能会一次性接收到这两条消息“ABCDEF”，也可能会分 3、4次才接收到这两条消息，如分 3 次分别接收“AB”、“CD”、“EF”，前面一次性接收就是**粘包**，后面分 3、4 接收就是**半包**。

## 粘包/半包主要原因
粘包的主要原因：
* 发送方每次写入数据 < 套接字缓冲区大小
* 接收方读取套接字缓冲区数据不够及时

半包的主要原因：
* 发送方每次写入数据 > 套接字缓冲区大小
* 发送的数据大于协议的 MTU（Maximum Transmission Unit，最大传输单元），必须拆包

从收发角度看：一次发送可能被多次接收（半包），多个发送可能被一次接收（粘包）
从传输角度看：一个发送可能占用多个传输包（半包），多个发送可能公用一个传输包（粘包）

**根本原因：TCP 协议是流式协议，消息无边界。**（UDP 像邮寄的包裹，虽然一次运输多个，但每个包裹都有“界限”，是一个一个签收的，所以使用 UDP 协议不会出现粘包/半包问题）

## 粘包/半包解决思路
要解决 TCP 粘包/半包问题，根本手段在于**找出消息边界**，下表列举了一些常见的解决方式。

方式\比较 | 寻找消息边界的方式 | 优点 | 缺点 | 推荐度
:-: | :-: | :-: | :-: | :-:
TCP 连接改成短连接，一个请求一个短连接 | 建立连接到释放连接之间的信息即为传输信息 | 简单 | 效率低下 | 不推荐
封装成帧（Framing）：固定长度 | 满足固定长度即可 | 简单 | 空间浪费 | 不推荐
封装成帧（Framing）：分隔符 | 分隔符之间 | 空间不浪费，也比较简单 | 内容本身出现分隔符时需转义，所以需要扫描内容 | 推荐
封装成帧（Framing）：固定长度字段存内容的长度信息 | 先解析固定长度的字段获取长度，然后读取后续内容 | 精确定位用户数据，内容也不用转义 | 长度理论上有限制，需提前预知可能的最大长度从而定义长度占用字节数 | 推荐
封装成帧（Framing）：其它方式 | 每种都不同，例如 JSON 可以看 {} 是否已经成对 | | |

## Netty 对三种常用封帧方式的支持
方式\支持 | 解码 | 编码
:-: | :-: | :-:
固定长度 | FixedLengthFrameDecoder | 简单
分隔符 | DelimiterBasedFrameDecoder | 简单
固定长度字段存内容的长度信息 | LengthFieldBasedFrameDecoder | LengthFieldPrepender

下面以“固定长度字段存内容的长度信息”这种方式来演示如何在 Netty 中使用：

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


