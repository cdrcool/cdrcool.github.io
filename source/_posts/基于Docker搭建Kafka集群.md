---
title: 基于 Docker 搭建 Kafka 集群
date: 2020-01-15 17:29:00
categories: Kafka
tags:
    - Docker
    - Docker Compose
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

3. 创建 kafka.yml
```yaml
version: '3.1'

services:
  kafka-1:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka-1
    container_name: kafka-1
    ports:
    - 9092:9092
    environment:
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://10.11.155.23:9092
      KAFKA_ZOOKEEPER_CONNECT: zk-1:2181,zk-2:2181,zk-3:2181
    external_links:
      - zk-1
      - zk-2
      - zk-3
    networks:
      default:
        ipv4_address: 172.22.0.14

  kafka-2:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka-2
    container_name: kafka-2
    ports:
    - 9093:9092
    environment:
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://10.11.155.23:9093
      KAFKA_ZOOKEEPER_CONNECT: zk-1:2181,zk-2:2181,zk-3:2181
    external_links:
      - zk-1
      - zk-2
      - zk-3
    networks:
      default:
        ipv4_address: 172.22.0.15

  kafka-3:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka-3
    container_name: kafka-3
    ports:
    - 9094:9092
    environment:
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://10.11.155.23:9094
      KAFKA_ZOOKEEPER_CONNECT: zk-1:2181,zk-2:2181,zk-3:2181
    external_links:
      - zk-1
      - zk-2
      - zk-3
    networks:
      default:
        ipv4_address: 172.22.0.16
      
networks:
  default:
    external:
      name: op_net
```

4. 启动 ZooKeeper 集群
```bash
docker-compose -f zookeeper.yml up -d
```

5. 启动 Kafka 集群
```bash
docker-compose -f kafka.yml up -d
```

参考资料：
1. [docker安装zookeeper集群](https://blog.csdn.net/qq_38270106/article/details/88789737)
1. [docker安装kafka集群](https://blog.csdn.net/qq_38270106/article/details/88809511)