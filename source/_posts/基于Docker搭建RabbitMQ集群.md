---
title: 基于 Docker 搭建 RabbitMQ 集群
date: 2020-01-27 18:30:00
categories: RabbitMQ
tags:
    - RabbitMQ
    - Docker
---
1. 创建 network
```bash
docker network create --driver bridge --subnet 172.22.0.0/16 --gateway 172.22.0.1  op_net
```

2. 创建 rabbitmq.yml
```yaml
version: '3.6'

services:
  rabbitmq-1:
    image: rabbitmq:3-management
    restart: always
    hostname: rabbitmq-1
    container_name: rabbitmq-1
    ports:
      - 5672:5672
      - 15672:15672
    environment:
      RABBITMQ_NODENAME: rabbitmq-1
      RABBITMQ_ERLANG_COOKIE: secret-cookie
    networks:
      default:
        ipv4_address: 172.22.0.17
  rabbitmq-2:
    image: rabbitmq:3-management
    restart: always
    hostname: rabbitmq-2
    container_name: rabbitmq-2
    ports:
      - 5673:5672
      - 15673:15672
    environment:
      RABBITMQ_NODENAME: rabbitmq-2
      RABBITMQ_ERLANG_COOKIE: secret-cookie
    networks:
      default:
        ipv4_address: 172.22.0.18
  rabbitmq-3:
    image: rabbitmq:3-management
    restart: always
    hostname: rabbitmq-3
    container_name: rabbitmq-3
    ports:
      - 5674:5672
      - 15674:15672
    environment:
      RABBITMQ_NODENAME: rabbitmq-3
      RABBITMQ_ERLANG_COOKIE: secret-cookie
    networks:
      default:
        ipv4_address: 172.22.0.19
      
networks:
  default:
    external:
      name: op_net
```

3. 启动 RabbitMQ 集群
```bash
docker-compose -f rabbitmq.yml up -d
```

4. 将 rabbitmq-2 添加到集群（RAM 类型）
```bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster --ram rabbitmq-1@rabbitmq-1
rabbitmqctl start_app
```

5. 将 rabbitmq-3 添加到集群
```bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster --ram rabbitmq-3@rabbitmq-3
rabbitmqctl start_app
```


参考资料：
1. [docker-compose配置rabbitmq集群服务器](https://blog.csdn.net/fqydhk/article/details/80624547)
2. [使用 docker-compose 安装搭建 RabbitMQ 集群](https://michael728.github.io/2019/06/07/docker-rabbitmq-env/)