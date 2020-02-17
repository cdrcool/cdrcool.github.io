---
title: Redis 数据结构
date: 2020-02-17 17:17:00
categories: Redis
---
Redis 中所有的数据结构都是以唯一的 key 字符串作为名称，然后通过这个唯一的 key 值来获取相应的 value 数据。不同类型的数据结构的差异就在于 value 的结构不一样。

## String（字符串）
String 类型是 Redis 的最基本的数据类型。

String 类型是二进制安全的。意思是 String 可以包含任何数据，比如 jpg 图片或者序列化的对象。

String 是动态字符串，内部结构实现上类似于 Java 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配（字符串实际分配的空间 capacity 一般要高于字符串实际长度 length）。当字符串长于小于 1M 时，扩容都是加倍现有的空间。如果超过 1M，扩容时一次只会多扩 1M 的空间。字符串的最大长度为 512M。

使用场景：
* 常规 key-value 缓存引用
* 常规计数：微博数、粉丝数

命令：
```
# 设置 key-value
set key value

# 获取 key 值
get key

# 检查是否存在 key
exists key

# 删除 key
del key

# 批量设置 key-value list
mset key1 value1 key2 value2 key3 value3

# 批量获取 keys
mget key1 key2 key3

# 设置 key 5s 后过期
expire key 5

# 设置 key-value，key 在 5s 后过期
setex key 5 value

# 如果 key 不存在，设置 key-value
setnx key value


# 如果 value 是一个整数，还可以对它进行自增/自减操作。
set age 30

# 自增 +1
incr age

# 自增 +5
incrby age 5

# 自减 -1
decr age

# 自减 -5
decrby age 5
```

## List（列表）
Redis List 是简单的字符串列表，按照插入顺序排序。可以添加一个元素到列表的头部（左边）或者尾部（右边）。

Redis List 相当于 Java 里面的 linkedList，注意它是链表而不是数组，这意味着 List 的插入和删除操作非常快，事件复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)。

当列表弹出了最后一个元素之后，该数据结构自动被删除，内存被回收。

使用场景：
* 消息队列
* 取最新 N 个数据的操作

命令：
```bash
# 在头部添加元素
lpush books java

# 在尾部添加元素
rpush books java

# 获取队列长度
llen books

# 获取所有元素（O(n) 慎用）
lrange books 0 -1

# 获取第 i 个元素（O(n) 慎用）
lindex books 0

# 保留第一个元素之后的值（O(n) 慎用）
ltrim books 1 -1

# 队列（右边进左边出）
rpush books python java golang

lpop books

lpop books

lpop books

# 栈（右边进右边出）
rpush books python java golang

rpop books

rpop books

rpop books
```

补充：
**Redis List 底层存储的不是一个简单的 linkedList，而是称之为快速链表 quicklist 的一个结构。**
在列表元素较少的情况下会使用一块连续的内存存储，这个结构是 ziplist，也即是压缩列表。它将所有的元素紧挨着一起存储，分配的是一块连续的内存。
当数据量比较多的时候就会改成 quicklist。
因为普通的链表需要用到附加指针（prev 和 next），会比较浪费空间，而且会加重内存的碎片化。所以 Redis 将链表和 ziplist 结合起来组成了 quicklist。也就是将多个 ziplist 使用双向指针串起来使用。这样既满足了快速的插入删除性能，又不会出现太大的空间冗余。

## Hash（字典）
Redis 的字典相当于 Java 语言里面的 HashMap，它是无序字典。内部实现结构上同 Java 的 HashMap 也是一致的：同样的数组 + 链表二维结构。第一维 hash 的数组位置碰撞时，就会将碰撞的元素使用链表串接起来。不同的是，Redis 的字典的值只能是字符串。

Redis 为了高性能，不能杜塞服务，所以采用了渐进式的 rehash 策略。渐进式 rehash 会在 rehash 的同时，保留新旧两个 hash 结构，查询时会同时查询两个 hash 结构，然后在后续的定时任务中以及 hash 操作指令中，循序渐进地将旧 hash 的内容一点点迁移到新的 hash 结构中。当迁移完成了，就会使用新的 hash 结构五而代之。

当 Hash 移除了最后一个元素之后，该数据结构会自动被删除，内存被回收。

Hash 结构特别适合用于存储对象。缺点在于其存储消耗要高于单个字符串。到底该使用 Hash 还是字符串，需要根据实际情况再三权衡。

使用场景：
* 存储部分变更数据，如用户信息等。

命令：
```bash
# put key-value
hset books java "think in java"

hset books golang "concurrency in go"

hset books python "python cookbook"

# 获取所有键值对
hgetall books

# 获取元素个数
hlen books

# 获取某个 key 的值
hget books java

# 设置某个 key 的值
hset books golang "learning go programming"

# 删除某个 key 的值
hdel books java

# 同字符串一样，hash 结构中的单个子 key 也可以进行计数
hset user age 30

hincrby user age 1
```

## Set（集合）
Redis Set 相当于 Java 语言里面的 HashSet，它内部的键值对是唯一且无序的。它的内部实现相当于一个特殊的字典，字典中所有的value 都是一个值 NULL。

当 Set 移除了最后一个元素之后，该数据结构会自动被删除，内存被回收。

使用场景：
* 交集，并集，差集
* 获取某段时间所有数据去重值

命令：
```bash
# 添加元素
sadd books python

# 重复添加
sadd books python

# 批量添加
sadd books java golang

# 查看集合成员（顺序不一定和插入的一致）
smembers books

# 获取元素个数
scard books

# 检查某个 value 是否在集合中
sismember books java

# 弹出一个元素
spop books

# 删除成员
srem books python
```

## zset（有序集合）
类似于 Java 的 SortedSet 和 HashMap 的集合体，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以给每个 value 赋予一个 score，代表这个 value 的排序权重。它的内部实现用的是一种叫做**跳跃链表**的数据结构。

zset 中最后一个 value 被移除后，数据结构自动删除，内存被回收。

使用场景：
* 排行榜应用，取 TOP N 操作
* 范围查找
* 优先级队列
* 需要精确设定过期时间的应用（score 值设置成过期时间的时间戳）

命令：
```bash
# 添加元素
zadd books 9.0 "think in java"
zadd books 8.9 "java concurrency"
zadd books 8.6 "java cookbook"

# 按 score 排序列出
zrange books 0 -1

# 按 score 逆序列出
zrevrange books 0 -1

# 获取元素个数
zcard books

# 获取指定元素的 score
zscore books "java concurrency"

# 获取指定元素的 score 排名
zrank books "java concurrency"

# 根据分值区间遍历 zset
zrangebyscore books 0 8.91

# 根据分值区间（(-=, 8.91]）遍历 zset，同时返回分值
zrangebyscore books -inf 8.91 withscores

# 删除成员
zrem books "java concurrency"
```

## BitMap（位图）
BitMap 就是通过一个 bit 位来表示某个元素对应的值或者状态, 其中的 key 就是对应元素本身，实际上底层也是通过对字符串的操作来实现。

使用场景：
* 用户签到
* 统计活跃用户
* 用户在线状态

命令：
```bash
# 零存零取
setbit w 1 1
getbit w 1

# 整存零取
set w hello
getbit w 1

# 统计指定范围内 1 的个数
bitcount w 0 -1

# 查找第一个 1 位
bitpos w 1

# ...
bitfield w ...
```

## HyperLogLog（基数统计）
Redis HyperLogLog 是用来做基数统计的算法。HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的。
在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。
但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。
另外，HyperLogLog 提供去重计数方案是不精确的，虽然不精确但是也不是非常不精确，标准误差是 0.81%，这样的精确度已经可以满足上面的 UV 统计需求了。

使用场景：
* 页面实时 UV
* 不适合单个用户的数据统计

命令：
```bash
# 添加计数
pfadd codehole user1

# 获取计数
pfcount codehole

# 添加计数
pfadd home user1

# 合并计数（并集计算）
pfmerge xx codehole home
```

## GEO（地理位置）
用于存储用户给定的地理位置信息，并对这些信息进行操作。GEO 数据结构总共有 6 个命令：geoadd、geopos、geodist、georadius、georadiusbymember。

## 容器类型数据结构的通用规则
list/set/hash/zset 这四种数据结构是容器型数据结构，它们共享下面两条通用规则：
* create if not exists：如果容器不存在，那就创建一个，再进行操作
* drop if no elements：如果容器里元数没有了，那么立即删除元数，释放内存。

## 过期时间
Redis 所有的数据结构都可以设置过期时间，时间到了，Redis 会自动删除相应的对象。

需要注意的是**过期是以对象为单位**，比如一个 hash 结构的过期是整个 hash 对象的过期，而不是其中的某个子 key。

还有一个需要特别注意的地方是**如果一个字符串已经设置了过期时间，然后调用了 set 方法修改了它，它的过期时间就会消失**。

## 参考资料
1. [Redis的8种数据类型](https://blog.csdn.net/hudeyong926/article/details/99540705)
2. [几率大的Redis面试题（含答案）](https://blog.csdn.net/Butterfly_resting/article/details/89668661)


