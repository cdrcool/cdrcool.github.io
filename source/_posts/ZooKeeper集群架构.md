---
title: ZooKeeper 集群架构
date: 2020-01-31 10:53:00
categories: ZooKeeper
---
![ZooKeeper集群架构](/images/zookeeper/ZooKeeper集群架构.png)

* 每个 Server 在内存中存储了一份数据
* ZooKeeper 启动时，将从实例中选举一个 leader（Paxos 协议）
* Leader 负责处理数据更新等操作（Zab 协议）
* 一个更新操作成功，当且仅当大多数 Server 在内存中成功修改数据

ZooKeeper 本身可以是单机模式，也可以是集群模式，为了 ZooKeeper 本身不出现单点故障，通常情况使用集群模式（Server 数目一般为奇数），而且是 master/slave 模式的集群。

### 角色
![ZooKeeper角色说明](/images/zookeeper/ZooKeeper角色说明.png)

#### Leader
Leader 不直接接受 Client 的请求，但接受由其他 Follower 和 Observer 转发过来的 Client 请求，此外，Leader 还负责投票的发起和决议，即时更新状态和数据。

#### Follower
Follower 角色接受客户端请求并返回结果，参与 Leader 发起的投票和选举，但不具有写操作的权限。

#### Observer
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