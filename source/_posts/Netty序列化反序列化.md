---
title: Netty 序列化/反序列化
date: 2020-02-08 13:20:00
categories: Netty
tags:
    - Netty
---
## 为什么需要二次编解码
假设我们把解决粘包半包问题的常用三种解码器叫做一次解码器，那么我们在项目中，除了可选的压缩解压缩之外，还需要一层解码，因为一次解码的结果是字节，需要和项目中所使用的对象做转换，方便使用，这层解码器可以成为“二次解码器”，相应的，对应的编码器是为了将 Java 对象转化成字节流方便存储或传输。

* 一次解码器：ByteToMessageDecoder
io.netty.buffer.ByteBuf（原始数据流） -> io.netty.buffer.ByteBuf（用户数据）

* 二次解码器：`MessageToMessageDecoder<I>`
io.netty.buffer.ByteBuf（用户数据） -> Java Object

## 常用二次编码码方式
* Java序列化：基本已经没人使用，因为它占用的空间比较大，且只有在 Java 中能够使用
* Marshaling
* XML：占用空间比较大
* JSON
* MessagePack
* ProtoBuf：灵活的、高效的用于序列化数据的协议；相较于 XML 和 JSON 格式，Protobuf 更小、更快、更便捷；Protobuf 是跨语言的，并且自带了一个编译器（protoc），只需要用它进行编译，可以自动生成 Java、Python、C++ 等代码，不需要再写其它代码。
* 其它

## 编解码方式选择依据
* 空间：编码后占用空间，需要比较不同的数据大小情况
* 时间：编解码速度，需要比较不同的数据大小情况
* 可读性
* 多语言的支持：重要
           
## Netty 对常用编解码方式的支持
1. 相关类都在包 io.netty.handler.codec
2. 支持 base64、bytes、compression、json、marshaling、protobuf、serialization、string、xml等
3. 编码器继承自 `MessageToMessageEncoder<I>`，解码器继承自 `MessageToMessageDecoder<I>`
