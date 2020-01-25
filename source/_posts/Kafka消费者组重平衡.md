---
title: Kafka 消费者组重平衡（Rebalance）
date: 2020-01-25 11:22:00
categories: Kafka
---
## 概念
Rebalance 本质上是一种协议，规定了一个 Consumer Group 下的所有 Consumer 如何达成一致，来分配订阅 Topic 的每个分区。在 Rebalance 过程中，所有 Consumer 实例共同参与，在协调者（Coordinator）组件的帮助下，完成订阅主题分区的分配。

## 触发条件
Rebalance 的触发条件有以下3个：
* 组成员数发生变更
比如有新的 Consumer 实例加入组或者离开组，抑或是有 Consumer 实例崩溃被“踢出”组。
* 订阅主题数发生变更
Consumer Group 可以是用正则表达式的方式订阅主题，比如 consumer.subscribe(Pattern.compile(“t.*c”)) 就表明该 Group 订阅所有以字母t开头、字母 c 结尾的主题。在 Consumer Group 的运行过程中，一旦我们新创建了一个满足这样条件的主题，那么该 Group 就会发生 Rebalance。
* 订阅主题的分区数发生变更
Kafka 当前只允许增加一个主题的分区数。当分区数增加时，就会触发订阅该主题的所有 Group 开启 Rebalance。

## 分区分配策略
Rebalance 发生时，Group 下所有的 Consumer 实例都会协调在一起共同参与。那每个 Consumer 实例怎么知道应该消费订阅主题的哪些分区呢？这就需要[Kafka 分区分配策略](https://cdrcool.github.io/2020/01/24/Kafka分区分配策略/)的协助了。

## 弊端
* Rebalance 影响 Consumer 端 TPS
这是因为在 Rebalance 过程中，所有 Consumer 实例都会停止消费，直到 Rebalance 完成。
* Rebalance 很慢
尤其是当我们的 Group 下成员很多的时候。
* Rebalance 效率不高
当前 Kafka 的设计机制决定了每次 Rebalance 时，Group 下的所有成员都要参与进来，而且通常不会考虑局部性原理，但局部性原理对提升系统性能是特别重要的。

## 避免 Rebalance
在真实的业务场景中，很多 Rebalance 都是计划外的或者说是不必要的。

要避免 Rebalance，还是要从 Rebalance 发生的时机入手。我们在前面说过，Rebalance 发生的时机有 3 个：
* 组成员数量发生变化
* 订阅主题数量发生变化
* 订阅主题的分区数发生变化
后面两个通常都是运维的主动操作，所以它们引发的 Rebalance 大都是不可避免的。

至于组成员数量发生变化，如果也是我们的主动操作，那么它也不属于我们要规避的那类“不必要 Rebalance”。


我们需要重点关注以下两类导致 Rebalance 的场景：

* Consumer 未能及时发送心跳
当 Consumer Group 完成 Rebalance 之后，每个 Consumer 实例都会定期地向 Coordinator 发送心跳请求，表明它还存活着。如果某个 Consumer 实例不能及时地发送这些心跳请求，Coordinator 就会认为该 Consumer已经“死”了，从而将其从 Group 中移除，然后开启新一轮 Rebalance。
Consumer 端有个参数，叫 **session.timeout.ms**，就是被用来表征此事的。该参数的默认值是10秒，即如果 Coordinator 在10秒之内没有收到 Group 下某 Consumer 实例的心跳，它就会认为这个 Consumer 实例已经挂了。可以这么说，session.timout.ms 决定了 Consumer 存活性的时间间隔。
除了这个参数，Consumer 还提供了一个允许我们控制发送心跳请求频率的参数，就是 **heartbeat.interval.ms**。这个值设置得越小，Consumer 实例发送心跳请求的频率就越高。频繁地发送心跳请求会额外消耗带宽资源，但好处是能够更加快速地知晓当前是否开启 Rebalance。

对于上面两个参数，在这里给出一些推荐数值，我们可以“无脑”地应用在你的生产环境中。
* 设置 session.timeout.ms = 6s。
* 设置 heartbeat.interval.ms = 2s。
* 要保证Consumer实例在被判定为“dead”之前，能够发送至少3轮的心跳请求，即 session.timeout.ms >= 3 * heartbeat.interval.ms。

* Consumer 消费时间过长
Consumer 端还有一个参数，用于控制 Consumer 实际消费能力对 Rebalance 的影响，即 **max.poll.interval.ms** 参数。它限定了 Consumer 端应用程序两次调用 poll 方法的最大时间间隔。它的默认值是 5 分钟，表示 Consumer 程序如果在 5 分钟之内无法消费完 poll 方法返回的消息，那么 Consumer 会主动发起“离开组”的请求，Coordinator 也会开启新一轮 Rebalance。
如果消费逻辑很重，此时，max.poll.interval.ms 参数值的设置显得尤为关键。如果要避免非预期的 Rebalance，我们最好将该参数值设置得大一点，比我们的下游最大处理时间稍长一点。这样，Consumer 就不会因为处理这些消息的时间太长而引发 Rebalance。

如果上面提到的 3 个参数，我们已经设置了合理的数值，却发现还是出现了 Rebalance，那么需要关注一下 Consumer 端的 **GC** 表现，比如是否出现了频繁的 Full GC 导致的长时间停顿，实际应用中经常出现因为GC设置不合理导致程序频发 Full GC 而引发的非预期 Rebalance。
