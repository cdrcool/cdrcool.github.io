---
title: Elasticsearch 读性能优化
date: 2020-02-21 12:29:00
categories: Elasticsearch
tags:
    - Elasticsearch
---
## 数据建模
* 尽可能 Denormalize 数据，从而获取最佳性能
    + 使用 Nested 类型的数据，查询数据会慢几倍
    + 使用 Parent/Child 关系，查询速度会慢几百倍
* 尽量将数据先行计算，然后保存在 Elasticsearch 中，以避查询时的 Script 计算
* 尽量使用 Filter Context，利用缓存机制，同时减少不必要的算分
* 结合 profile、explain API 分析慢查询的问题，持续优化数据模型
    + 严禁使用 * 开头的通配符 Terms 查询
* 聚合查询时，控制聚合的数量，以减少内存的开销

## 优化分片
* 避免 Over Sharing
一个查询需要访问每一个分片，分片过多，会导致不必要的查询开销
* 结合应用场景，控制单个分片的尺寸
Search: 200GB Logging: 400GB
* Force-merge Read-only 索引
使用基于时间序列的索引，将只读的索引进行 force merge，减少 segment 数量
