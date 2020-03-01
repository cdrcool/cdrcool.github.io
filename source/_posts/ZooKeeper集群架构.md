---
title: ZooKeeper 集群架构
date: 2020-01-31 10:53:00
categories: ZooKeeper
tags:
    - ZooKeeper
---
![ZooKeeper集群架构](/images/zookeeper/ZooKeeper集群架构.png)

* 每个 Server 在内存中存储了一份数据
* ZooKeeper 启动时，将从实例中选举一个 leader（Paxos 协议）
* Leader 负责处理数据更新等操作（Zab 协议）
* 一个更新操作成功，当且仅当大多数 Server 在内存中成功修改数据

ZooKeeper 本身可以是单机模式，也可以是集群模式，为了 ZooKeeper 本身不出现单点故障，通常情况使用集群模式（Server 数目一般为奇数），而且是 master/slave 模式的集群。

## 角色
![ZooKeeper角色说明](/images/zookeeper/ZooKeeper角色说明.png)

### Leader
Leader 不直接接受 Client 的请求，但接受由其他 Follower 和 Observer 转发过来的 Client 请求，此外，Leader 还负责投票的发起和决议，即时更新状态和数据。

### Follower
Follower 角色接受客户端请求并返回结果，参与 Leader 发起的投票和选举，但不具有写操作的权限。

### Observer
Observer 角色接受客户端连接，将写操作转给 Leader，但 Observer 不参与投票（即不参加一致性协议的达成），只同步 Leader 节点的状态，Observer 角色是为集群系统扩展而生的。

应用场景：
* 提高集群的读性能（未参与事务的提交过程）
* 跨数据中心部署

### Follower vs Observer 请求流程
Follower-Leader 写请求流程：
![Follower-Leader请求流程](/images/zookeeper/Follower-Leader请求流程.png)

1. 节点 1（Follower） 收到写请求，转发到节点 2（Leader）
2. 节点 2 发送 Propose 给集群中所有 Follower 节点
3. Follower 节点收到 Propose 后，返回 Accept 给 Leader
4. Leader 收到大多数节点的 Accept 后，向所有节点发送 Commit
5. 节点 1 收到 Commit 后，返回客户端，告诉客户端写成功

Observer-Leader 写请求流程：
![Obeserver-请求流程](/images/zookeeper/Obeserver-Leader请求流程.png)

1. 节点 1（Observer） 收到写请求，转发到节点 2（Leader）
2. 节点 2 发送 Propose 给集群中所有 Follower 节点
3. Follower 节点收到 Propose 后，返回 Accept 给 Leader
4. Leader 收到大多数节点的 Accept 后，向所有节点发送 Commit
5. 节点 1 不参与事务的提交过程，而是一直等待，等到收到 Leader 的 INFORM 后，返回客户端，告诉客户端写成功

## 节点数
ZooKeeper 要求集群节点数大于 1，且为单数，原因有以下两点。

### 容错
由于在增删改操作中需要半数以上服务器通过，来分析以下情况：
* 2 台服务器，至少 2 台正常运行才行（2 的半数为 1，半数以上最少为 2），正常运行 1 台服务器都不允许挂掉
* 3 台服务器，至少 2 台正常运行才行（3 的半数为 1.5，半数以上最少为 2），正常运行可以允许 1 台服务器挂掉
* 4 台服务器，至少 3 台正常运行才行（4 的半数为 2，半数以上最少为 3），正常运行可以允许 1 台服务器挂掉
* 5 台服务器，至少 3 台正常运行才行（5 的半数为 2.5，半数以上最少为 3），正常运行可以允许 2 台服务器挂掉
* 6 台服务器，至少 3 台正常运行才行（6 的半数为 3，半数以上最少为 4），正常运行可以允许 2 台服务器挂掉

通过以上可以发现，3 台服务器和 4 台服务器都最多允许 1 台服务器挂掉，5 台服务器和 6 台服务器都最多允许 2 台服务器挂掉，但是明显 4 台服务器成本高于 3 台服务器成本，6 台服务器成本高于 5 服务器成本。这是由于半数以上投票通过决定的。

### 防脑裂
一个 ZooKeeper 集群中，可以有多个 follower、observer 服务器，但是必需只能有一个 leader 服务器。如果 leader 服务器挂掉了，剩下的服务器集群会通过半数以上投票选出一个新的 leader 服务器。

集群互不通讯情况：
* 一个集群 3 台服务器，全部运行正常，但是其中 1 台裂开了，和另外 2 台无法通讯。3 台机器里面 2 台正常运行过半票可以选出一个 leader。
* 一个集群 4 台服务器，全部运行正常，但是其中 2 台裂开了，和另外 2 台无法通讯。4 台机器里面 2 台正常工作没有过半票以上达到3，无法选出 leader 正常运行。
* 一个集群 5 台服务器，全部运行正常，但是其中 2 台裂开了，和另外 3 台无法通讯。5 台机器里面 3 台正常运行过半票可以选出一个 leader。
* 一个集群 6 台服务器，全部运行正常，但是其中 3 台裂开了，和另外 3 台无法通讯。6 台机器里面 3 台正常工作没有过半票以上达到 4，无法选出 leader 正常运行。

通可以上分析可以看出，为什么 ZooKeeper 集群数量总是单出现，主要原因还是在于第 2 点，防脑裂，对于第 1 点，无非是正常控制，但是不影响集群正常运行。但是出现第 2 种裂的情况，ZooKeeper 集群就无法正常运行了。