---
title: Redis 布隆过滤器
date: 2020-02-19 20:05:00
categories: Redis
---
## 概念
布隆过滤器（Bloom Filter）是一种空间效率极高的概率型算法和数据结构，主要用来判断一个元数是否在集合中存在。
因为它是一个概率型的算法，所以会存在一定的误差，如果传入一个值去布隆过滤器中检索，可能会出现检测存在的结果实际上是不存在的，但是肯定不会出现实际上不存在却反馈存在的结果。
因此，布隆过滤器不适合那些“零错误”的应用场合。而在能容忍低错误率的应用场合下，布隆过滤器通过极少的错误换取了存储空间的极大节省。

## 使用
我们需要通过加载 module 来使用 Redis 中的布隆过滤器。使用 Docker 可以直接在 Redis 中体验布隆过滤器。 
```bash
docker run -d -p 6379:6379 --name bloomfilter redislabs/rebloom
docker exec -it bloomfilter redis-cli
```

Redis 布隆过滤器主要就 2 个指令:
* bf.add 添加元素到布隆过滤器中：`bf.add urls https://cdrcool.github.io`。
* bf.exists 判断某个元素是否在过滤器中：`bf.exists urls https://cdrcool.github.io`。

如果我们需要一次添加或查询多个元素，还可以使用以下 2 个指令：
* bf.madd 判断某个元素是否在过滤器中：`bf.exists urls https://cdrcool.github.io https://github.com`。
* bf.mexists 判断某个元素是否在过滤器中：`bf.exists urls https://cdrcool.github.io https://github.com`。

当我们调用 bd.add 或 bf.madd 指令时，如果此时布隆过滤器不存在，那么 Redis 会帮我们自动创建。Redis 其实还提供了自定义参数的布隆过滤器，需要我们在 add 之前使用 bf.reserve 指令显示创建，如：`bf.reserve urls 0.01 100`。
bf.reservee 有 3 个参数：
* key 过滤器的名字。
* error_rate 允许布隆过滤器的错误率，这个值越低过滤器的位数组的大小越大，占用空间也就越大。
* initial_size 布隆过滤器可以储存的元素个数，当实际存储的元素个数超过这个值之后，过滤器的准确率会下降。

需要注意的是，在执行 bf.reservee 这个命令之前过滤器的名字应该不存在，不然会报错。

## 原理
每个布隆过滤器对应到 Redis 的数据结构里面就是一个大型的 BitMap 和几个不一样的无偏 hash 函数。所谓无偏就是能够把元素的 hash 值算的比较均匀

向布隆过滤器中添加 key 时，会使用多个 hash 函数对 key 进行 hash 算的一个整数索引值然后对位数组长度进行取模运算得到一个位置，每个 hash 函数都会算的一个不同的位置，再把位数组的这几个位置都置为 1 就完成了 add 操作。

向布隆过滤器询问 key 是否存在时，跟 add 一样，也会把 hash 的几个位置都算出来，看看位数组中这几个位置是否都为 1，只要有一个位为 0，那么说明布隆过滤器中这个 key 不存在。如果都是 1，这并不能说明这个 key 就一定存在，只是极有可能存在，因为这些位置被置为 1 可能是因为其它的 key 存在所致。如果这个数组比较稀疏，判断正确的概率就会很大。

使用时不要让实际元素远大于初始化大小，当实际元素开始超出初始化大小时，应该对布隆过滤器进行重建，重新分配一个 size 更大的过滤器，再将所有的历史元素批量 add 进去(这就要求我们在其它的存储器中记录所有的历史元素)。

## 应用场景
* 爬虫系统中 URL 去重
* 邮箱系统中垃圾邮件过滤
* 推荐系统中新闻推荐 

## 参考资料
1. [Redis 中的布隆过滤器](https://segmentfault.com/a/1190000016721700)