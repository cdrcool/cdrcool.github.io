---
title: RabbitMQ 保证消息的顺序性
date: 2020-01-28 16:55:00
categories: RabbitMQ
---
同一个 Queue，如果有多个 Consumer，那么是做不到消息的有序性的。
因此，我们可以基于同一规则（对唯一标识进行 hash），将单个 Queue 拆分成多个 Queue，再将拆分后的 Queue 与 Consumer 一一对应。
同时，我们应该关闭 auto ack，改为 manual ack，并设置 prefetchCount = 1（默认），即 channel.basicQos(1)。