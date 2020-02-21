---
title: Elasticsearch 写性能优化
date: 2020-02-21 11:25:00
categories: Elasticsearch
---
## 客户端
* 多线程写入
* 使用 Bulk API 进行批量写
    + 单个 Bulk 请求体的数据量不要太大，官方建议大约 5-15 mb
    + 写入端的 bulk 请求超时需要足够长，建议 60s 以上
    + 写入端尽量将数据轮询达到不同的节点上 

## 服务器端
* 降低 IO 操作
    + 使用 ES 自动生成文档的 id
    + 设置 ES 配置（如 refresh_interval）

* 降低 CPU 和存储开销
    + 减少不必要的分词
    + 避免不必要的 doc_values（禁用后不能用于排序、聚合以及脚本访问）
    + 文档的字段尽量保证相同的顺序（提高文档的压缩率）
    
* 尽可能做到写入和分片的均衡负载，实现水平扩展
    + 使用 Shard Filtering 来控制将分配到哪个节点
    + Write Load Balancer
    + 配置 index.routing.allocation.total_share_per_node，限定每个索引在每个节点上可分配的主分片数

* 调整 Bulk 线程池和队列
    + 应使用固定大小（CPU 核数 + 1）的线程池，避免过多的上下文切换
    + 队列大小可以适当增加，不要过大，否则占用的内存会成为 GC 的负担

## 文档建模（关闭无关的功能）
* 只需要聚合不需要搜索，index 设置成 false
* 不需要算分，norms 设置成 false
* 对于指标型数据，关闭 _source，以减少 IO 操作
* 对于 Text 类型字段，不采用默认的 mapping，应结合实际需求人工设定
* 不要对字符串使用默认的 dynamic mapping（字段数量过多，会对性能产生较大的影响）
* 合理设置 index_options: doc |freqs | positions | offsets。（Text 类型默认记录级别为 positions，其它类型默认为 docs。记录内容越多，占用的存储空间越大。）

## 性能取舍
如果需要追求极致的写入速度，可以牺牲数据可靠性及搜索实时性以换取性能。
* 牺牲可靠性：将副本分片设置为 0，写入完毕再调整回去
* 牺牲搜索实时性：增加 refresh_interval 的时间，默认 1s；增大 indices.memory.index_bufer_size，默认是 10%，会导致自动触发 refresh
* 牺牲可靠性：修改 translog 的配置，默认每次请求都会异步落盘
