---
title: 基于 Docker 搭建 ZooKeeper 集群
date: 2020-01-31 18:05:00
categories: ZooKeeper
tags:
    - Docker
---
1. 创建 network
```bash
docker network create --driver bridge --subnet 172.22.0.0/16 --gateway 172.22.0.1  op_net
```

2. 创建 zookeeper.yml
```yaml
version: '3.1'

services:
  zk-1:
    image: zookeeper
    restart: always
    hostname: zk-1
    container_name: zk-1
    ports:
      - 2181:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888 server.2=zk-2:2888:3888 server.3=zk-3:2888:3888
    networks:
      default:
        ipv4_address: 172.22.0.11

  zk-2:
    image: zookeeper
    restart: always
    hostname: zk-2
    container_name: zk-2
    ports:
      - 2182:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zk-1:2888:3888 server.2=0.0.0.0:2888:3888 server.3=zk-3:2888:3888;
    networks:
      default:
        ipv4_address: 172.22.0.12

  zk-3:
    image: zookeeper
    restart: always
    hostname: zk-3
    container_name: zk-3
    ports:
      - 2183:2181
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zk-1:2888:3888 server.2=zk-2:2888:3888 server.3=0.0.0.0:2888:3888
    networks:
      default:
        ipv4_address: 172.22.0.13
      
networks:
  default:
    external:
      name: op_net
```

3. 启动 ZooKeeper 集群
```bash
docker-compose -f zookeeper.yml up -d
```


参考资料：
1. [docker安装zookeeper集群](https://blog.csdn.net/qq_38270106/article/details/88789737)