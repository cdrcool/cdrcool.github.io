---
title: ZooKeeper Watcher 机制
date: 2020-02-02 22:54:00
categories: ZooKeeper
---
ZooKeeper 提供了分布式数据发布/订阅功能，一个典型的发布/订阅模型系统定义了一种一对多的订阅关系，能让多个订阅者同时监听某一个主题对象，当这个主题对象自身状态变化时，会通知所有订阅者，使它们能够做出相应的处理。
ZooKeeper 中，引入了 Watcher 机制来实现这种分布式的通知功能。ZooKeeper 允许客户端向服务端注册一个 Watcher 监听，当服务端的一些事件触发了这个 Watcher，那么就会向指定客户端发送一个事件通知来实现分布式的通知功能。触发事件种类很多，如：节点创建，节点删除，节点改变，子节点改变等。
总的来说可以概括 Watcher 为以下三个过程：客户端向服务端注册 Watcher、服务端事件发生触发 Watcher、客户端回调 Watcher 得到触发事件情况。

Zookeeper中所有读操作（exists(),getData(),getChildren()）都可以设置 Watch 选项。不同的观察能被触发事件不同:
* 当所观察的 znode 被创建、删除或其数据被更新时，设置在 exists 操作上的观察将被触发。
* 当所观察的 znode 被删除，或其数据被更新时，设置在 getData 操作上的观察将被触发。创建 znode 不会触发 getData 操作上的观察，因为 getData 操作成功执行的前提时 znode 必须已经存在。
* 所观察的 znode 的一个子节点被创建或被删除的时候，或者所观察的 znode 自己被删除的时候，设置在 getChildren 操作上的观察将被触发，可以通过观察事件的类型来判断被删除的是 znode 还是其子节点；NodeDelete 类型代表 znode 被删除，NodeChildrenChanged 类型代表一个子节点被删除。

New ZooKeeper 时注册的 Watcher 叫 default watcher，它不是一次性的，只对 client 的连接状态变化作出反应。

那么要实现 Watch，就必须实现 org.apache.zookeeper.Watcher 接口，并且将实现类的对象传入到可以 Watch 的方法中。
在上述说道的所有读操作中，如果需要 Watcher，我们可以自定义 Watcher，如果是 Boolean 型变量，当为 true 时，则使用系统默认的 Watcher，系统默认的 Watcher 是在 Zookeeper 的构造函数中定义的 Watcher。参数中 Watcher 为空或者 false，表示不启用 Wather。

## Watch 机制特点
* 一次性触发
事件发生触发监听，一个 Watcher event 就会被发送到设置监听的客户端，这种效果是**一次性的**，后续再次发生同样的事件，不会再次触发。如果还需要关注数据的变化，需要再次注册 Watcher。
* 事件封装
ZooKeeper 使用 WatchedEvent 对象来封装服务端事件并传递。WatchedEvent 包含了每一个事件的三个基本属性：通知状态（keeperState），事件类型（EventType）和节点路径（path）。
* event 异步发送
watcher 的通知事件从服务端发送到客户端是异步的。
* 先注册再触发
Zookeeper 中的 watch 机制，必须客户端先去服务端注册监听，这样事件发送才会触发监听，通知给客户端。

## 常用操作
操作 | 描述
:-: | :-:
create | 创建一个 znode（必须要有父节点）
delete | 删除一个 znode（该 znode 不能有任何子节点）
exists | 测试一个 znode 是否存在并且查询它的元数据
getData,setdata | 获取/设置一个 znode 所保存的数据
getChildren | 获取一个 znode 的子节点列表
getACL,setACL | 获取/设置一个 znode 的 ACL
sync | 将客户端的 znode 视图与 Zookeeper 同步（将当前客户端连接上的 znode 数据与主节点同步，将数据更新到最新）

注意：
* 在使用 delete 或 setData 操作时必须提供被更新的版本号（可以通过 exists 操作获得）。
* Zookeeper 的写操作是具有原子性的，在写操作没有完成前，Zookeeper 允许客户端读到的数据之后于 Zookeeper 服务的最新状态。
* Zookeeper 中有一个被称为 multi 的操作，用于将多个基本操作集合组成一个操作单元，并确保这些基本操作同时被成功执行，或者同时失败。

## 事件和状态
同一个事件类型在不同的通知状态中代表的含义有所不同，下表列举了常见的通知状态和事件类型。

KeeperState | EventType | 触发条件 | 说明
:-: | :-: | :-: | :-:
 | None（-1） | 客户端与服务端成功建立连接 | 
SyncConnected（0） | NodeCreated（1） | Watcher 监听的对应数据节点被创建 | 
 | NodeDeleted（2） | Watcher 监听的对应数据节点被删除 | 此时客户端和服务器处于连接状态
 | NodeDataChanged（3） | Watcher 监听的对应数据节点的数据内容发生变更 | 
 | NodeChildChanged（4） | Wather 监听的对应数据节点的子节点列表发生变更 | 
Disconnected（0） | None（-1） | 客户端与 ZooKeeper 服务器断开连接 | 此时客户端和服务器处于断开连接状态
Expired（-112） | Node（-1） | 会话超时	此时客户端会话失效，通常同时也会受到 SessionExpiredException 异常
AuthFailed（4） | None（-1） | 通常有两种情况，1：使用错误的schema 进行权限检查 2：SASL 权限检查失败 | 通常同时也会收到 AuthFailedException 异常

虽然说 Zookeeper 需要先注册再触发，但是连接状态事件(type=None, path=null)不需要客户端注册，客户端只要有需要直接处理就行了。

参考资料：
[zookeeper的watcher机制](https://blog.csdn.net/xxydzyr/article/details/93390162)
[zookeeper（四）：核心原理（Watcher、事件和状态）](https://www.cnblogs.com/shamo89/p/9787176.html)