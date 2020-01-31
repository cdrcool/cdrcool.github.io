---
title: ZooKeeper 数据模型
date: 2020-01-31 10:52:00
categories: ZooKeeper
---
ZooKeeper 的数据模型是**层次模型**。层次模型常见于文件系统。层次模型和 key-value 模型是两种主流的数据模型。ZooKeeper 使用文件系统模型主要基于以下两点考虑：
1. 文件系统的树形结构便于表达数据之间的层次关系
2. 文件系统的属性结构便于为不同的应用分配独立的命名空间（namespace）

![ZooKeeper数据模型](/images/zookeeper/ZooKeeper数据模型.png)

ZooKeeper 的层次模型称作 Data tree。Data tree 的每个节点叫做 znode。不同于文件系统，每个节点都可以保存数据。每个节点都有一个版本（version）。版本从 0 开始计数。

## 节点类型
一个 znode 可以是持久性的，也可以是临时性的；可以是顺序性的，也可以是非顺序性的。
1. PERSISTENT
持久性的 znode，在创建之后即使发生 ZooKeeper 集群宕机或者 client 宕机也不会丢失。
2. EPHEMERAL
临时性的 znode，client 宕机或者 client 在指定的 timeout 时间内没有给 ZooKeeper 集群发消息，这样的节点就会消失。
3. PERSISTENT_SEQUENTIAL
持久顺序性的 znode，除了具备持久性 znode 的特点之外，znode 的名字具备顺序性。
4. EPHEMERAL_SEQUENTIAL
临时顺序性的 znode，除了具备临时性 znode 的特点之外，znode 的名字具备顺序性。

每一个顺序性的 znode 关联一个唯一的单调递增整数。这个单调递增整数是 znode 名字的后缀。

其实除了以上 4 中节点类型，还有另外一类节点：container 节点。
container 节点是一种新引入的 znode，目的在于下挂子节点。当一个 container 节点的所有子节点被删除之后，ZooKeeper 会删除掉这个 container 节点。服务发现的 base path 节点和服务节点就是 container 节点。

## Data tree接口
ZooKeeper 对外提供一个用来访问 Data tree 的简化文件系统 API：
* 使用 UNIX 风格的路径名来定位 znode，例如 /A/X 表示 znode A 的子节点 X。
* znode 的数据只支持全量写入和读取，没有像通用文件系统那样支持部分写入和读取。
* Data tree 的所有 API 都是 wait-free 的。正在执行中的 API 调用不会影响其他 API 的完成。
* Data tree 的 API 都是对文件系统的 wait-free 操作，不直接提供锁这样的分布式协同机制。但是 Data tree 的 API 非常强大，可以用来实现多种分布式协同机制。