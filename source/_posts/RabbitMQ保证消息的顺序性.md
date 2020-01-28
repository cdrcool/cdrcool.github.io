---
title: RabbitMQ 保证消息的顺序性
date: 2020-01-28 16:55:00
categories: RabbitMQ
---
## 方案一
* 一个 Queue 对应一个 Consumer
* 关闭 auto ack, 设置为 manual ack
* 设置 prefetchCount = 1，即 channel.basicQos(1)，默认即为 1

## 方案二
消息实体中增加：版本号 & 状态机 & msgid & parent_msgid，通过 parent_msgid 判断消息的顺序（需要全局存储，记录消息的执行状态）。