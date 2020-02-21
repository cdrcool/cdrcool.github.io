---
title: ZooKeeper Leader 选举
date: 2020-02-03 19:21:00
categories: ZooKeeper
---
Leader 选举是保证分布式数据一致性的关键所在。当 ZooKeeper 集群中的一台服务器出现以下两种情况之一时，需要进入 Leader 选举。
1. 服务器初始化启动（集群的每个节点都没有数据 -> 以 SID 的大小为准）
2. 服务器运行期间无法和 Leader 保持连接。（集群的每个节点都有数据，或者 Leader 宕机 -> 以 ZXID 和 SID 的最大值为准）

## Leader 选举示例
### 服务器启动时期的 Leader选举
若进行 Leader 选举，则至少需要 2 台机器，两台的高可用性会差一些，如果 Leader 宕机，就剩下一台，自己没有办法选举。这里选取 3 台机器组成的服务器集群为例。

在集群初始化阶段，当有一台服务器 Server1 启动时，其单独无法进行和完成 Leader 选举，当第二台服务器 Server2 启动时，此时两台机器可以相互通信，每台机器都试图找到 Leader，于是进入 Leader 选举过程。选举过程如下：
1. **每个 Server 发出一个投票。** 由于是初始情况，Server1 和 Server2 都会将自己作为 Leader 服务器来进行投票，每次投票都会包含所推举的服务器的 myid 和 ZXID，使用（myid, ZXID）来表示，此时 Server1 的投票为 (1,0)，Server2 的投票为 (2,0)，然后各自将这个投票发给集群中其他机器。
2. **接收来自各个服务器的投票。** 集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自 LOOKING 状态的服务器。
3. **处理投票。** 针对每一个投票，服务器都需要将别人的投票额自己的投票进行 PK，PK 规则如下：
    * **优先检查ZXID。** ZXID 比较大的服务器优先作为 Leader。（**这个很重要：是数据最近原则，保证数据的完整性**）
    * **如果 ZXID 相同，那么就比较 myid。** myid 较大的服务器作为 Leader 服务器。（集群节点标识）
对于 Server1 而言，它的投票是 (1,0)，接收 Server2 的投票为 (2,0)。首先会比较两者的 ZXID，均为 0。在比较 myid，此时 Server2 的 myid 最大，于是更新自己的投票为 (2, 0)，然后重新投票。对于 Server2 而言，其无需更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。
4. **统计投票。** 每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接收到相同的投票信息，对于 Server1、Server2 而言，都统计出集群中已经有两台机器接收了 (2,0) 的投票西悉尼，此时便认为已经选出了 Leader。
5. **改变服务器状态。** 一旦确定了 Leader，每个服务器就会更新自己的状态，如果是 Follower，那么就变更为 FOLLOWING，如果是 Leader，就变更为 LEADING。

### 服务器运行时期的 Leader 选举
在 ZooKeeper 运行期间，Leader 与非 Leader 服务器各司其职，即便当有非 Leader 服务器宕机或新加入，此时也不会影响 Leader，但是一旦 Leader 服务器挂了，那么整个集群将暂停对外服务，进入新一轮 Leader 选举，其过程和启动时期的 Leader 选举过程基本一致。

假设正在运行的有 Server1、Server2、Server3 三台服务器，当前 Leader 是 Server2，若某一时刻 Leader 挂了，此时便开始 Leader 选举。选举过程如下：
1. **变更状态。** Leader 挂掉后，余下的 Observer 服务器都会将自己的服务器状态变更为 LOOKING，然后开始进入 Leader 选举过程。
2. **每个 Server 会发出一个投票。** 在运行期间，每个服务器上的 ZXID 可能不同，此时假定 Server1 的 ZXID 为 123，Server3 的 ZXID 为 122，在第一轮投票中，Server1 和 Server3 都会投自己，产生投票 (1, 123) 和 (3, 122)，然后各自将投票发送给集群中所有机器。
3. **接收来自各个服务器的投票。** 与启动时过程相同。
4. **处理投票。** 与启动时过程相同，此时，Server1 将会成为 Leader。
5. **统计投票。** 与启动时过程相同。
6. **改变服务器的状态。** 与启动时过程相同。

### FinalizeWait
* Happy Case
![Leader选举-happycase](/images/zookeeper/Leader选举-happycase.png)

* Unhappy Case
![Leader选举-unhappycase](/images/zookeeper/Leader选举-unhappycase.png)

在这种情况之下，node3 不会响应来自 node2 的请求，node2 会在 timeout 之后重试。node2 在 timeout 之前没有办法处理请求。

* Unhappy Case & FinalizeWait
![Leader选举-finalizewait](/images/zookeeper/Leader选举-finalizewait.png)

一个节点在获得一个 vote 的 quorum 之后，在完成选举之前会等待一段时间。如果在这段时间收到更新的 vote，继续执行选举算法。

## Leader 选举算法分析
在 3.4.0 后的 ZooKeeper 的版本只保留了 TCP 版本的 FastLeaderElection 选举算法。当一台机器进入 Leader 选举时，当前集群可能会处于以下两种状态：集群中已经存在 Leader、集群中不存在 Leader。
对于集群中已经存在 Leader 而言，此种情况一般都是某台机器启动得较晚，在其启动之前，集群已经在正常工作，对这种情况，该机器试图去选举 Leader 时，会被告知当前服务器的 Leader 信息，对于该机器而言，仅仅需要和 Leader 机器建立起连接，并进行状态同步即可。
而在集群中不存在 Leader 情况下则会相对复杂，其步骤如下：
1. **第一次投票。** 无论哪种导致进行 Leader 选举，集群的所有机器都处于试图选举出一个 Leader 的状态，即 LOOKING 状态，LOOKING 机器会向所有其它机器发送消息，该消息成为投票。投票中包含了 SID（服务器的唯一标识）和 ZXID （事务ID），(SID, ZXID) 形式来标识一次投票信息。
假定 ZooKeeper 由 5 台机器组成，SID 分别为 1、2、3、4、5，ZXID 分别为 9、9、9、8、8，并且此时 SID 为 2 的机器是 Leader 机器，某一时刻，1、2 所在机器出现故障，因此集群开始进行 Leader 选举。
在第一次投票时，每台机器都会将自己作为投票对象，于是 SID 为 3、4、5 的机器投票情况分别为 (3,9)、(4,8)、(5,8)。
2. **变更投票。** 每台机器发出投票后，也会收到其它机器的投票，每台机器会根据一定规则来处理收到的其它机器的投票，并以此来决定是否需要变更自己的投票，这个规则也是整个 Leader 选举算法的核心所在，其中的术语描述如下：
    * vote_sid 接收到的投票中锁推举的 Leader 服务器的 SID。
    * vote_zxid 接收到的投票中所推举 Leader 服务器的 ZXID。
    * self_sid 当前服务器自己的 SID。
    * self_zxid 当前服务器自己的 ZXID。
每次对收到的投票的处理，都是对 (vote_sid,vote_zxid) 和 (self_sid, self_zxid) 对比的过程。
    规则一：如果 vote_zxid 大于 self_zxid，就认可当前收到的投票，并再次将该投票发送出去。
    规则二：如果 vote_zxid 小于 self_zxid，那么坚持自己的投票，不做任何变更。
    规则三：如果 vote_zxid 等于 self_zxid，那么久对比两者的 SID，如果 vote_sid 大于 self_sid，那么就认可当前收到的投票，并再次将该投票发送出去。
    规则四：如果 vote_zxid 等于 self_zxid，并且 vote_sid 小于 self_sid，那么坚持自己的投票，不做任何变更。
综合上面的规则，给出下面的集群变更过程。
![Leader选举过程](/images/zookeeper/Leader选举过程.png)
3. **确定 Leader。** 经过第二轮投票后，集群中的每台机器都会再次接收到其它机器的投票，然后开始统计投票，如果一台机器收到了超过半数的相同投票，那么这个投票对应的 SID 的机器即为 Leader。此时 Server3 将成为 Leader。

## Leader 选举实现细节
1. 服务器状态
服务器具有四种状态，分别是 **LOOKING、FOLLOWING、LEADING、OBSERVING**。

* LOOKING 寻找 Leader状态。当服务器处于该状态时，它会认为当前集群中没有 Leader，因此需要进入 Leader 选举状态。
* FOLLOWING 跟随者状态。表明当前服务器角色是 Follower。
* LEADING 领导者状态。表明当前服务器角色是 Leader。
* OBSERVING 观察者状态。表明当前服务器角色是 Observer。

2. 投票数据结构
每个投票中包含了两个最基本的信息，所推举服务器的 SID 和 ZXID，投票（Vote）在 ZooKeeper 中包含字段如下：
* id 被推举的 Leader 的 SID。
* zxid 被推举的 Leader 的事务 ID。
* electionEpoch 逻辑时钟，用来判断多个投票是否在同一轮选举周期中，该值在服务器端是一个自增序列，每次进入新一轮的投票中，都会对该值进行加 1 操作。
* peerEpoch 被推举的 Leader 的 epoch。
* state 当前服务器的状态。

## 参考资料
1. [Zookeeper选举算法原理](https://www.cnblogs.com/sweet6/p/10574574.html)