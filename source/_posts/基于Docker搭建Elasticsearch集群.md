---
title: 基于 Docker 搭建 Elasticsearch 集群
date: 2020-02-11 11:10:00
categories: Elasticsearch
tags:
    - Elasticsearch
    - Docker
---
1. 创建 network
```bash
docker network create --driver bridge --subnet 172.22.0.0/16 --gateway 172.22.0.1  op_net
```

2. 创建 elasticsearch.yml
```yaml
version: '3.6'

services:
  cerebro:
    image: lmenezes/cerebro
    restart: always
    container_name: cerebro
    ports:
      - 9000:9000
    command:
      - -Dhosts.0.host=http://elasticsearch:9200
    networks:
      default:
        ipv4_address: 172.22.0.24
  kibana:
    image: kibana
    restart: always
    container_name: kibana
    environment:
      - I18N_LOCALE=zh-CN
      - TIMELION_ENABLED=true
      - XPACK_GRAPH_ENABLED=true
      - XPACK_MONITORING_COLLECTION_ENABLED="true"
    ports:
      - 5601:5601
    networks:
      default:
        ipv4_address: 172.22.0.25
  elasticsearch:
    image: elasticsearch
    restart: always
    container_name: es_hot
    environment:
      - cluster.name=es_cluster
      - node.name=es_hot
      - node.attr.box_type=hot
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.seed_hosts=es_hot,es_warm,es_cold
      - cluster.initial_master_nodes=es_hot,es_warm,es_cold
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es_data_hot:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      default:
        ipv4_address: 172.22.0.21
  elasticsearch2:
    image: elasticsearch
    restart: always
    container_name: es_warm
    environment:
      - cluster.name=es_cluster
      - node.name=es_warm
      - node.attr.box_type=warm
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.seed_hosts=es_hot,es_warm,es_cold
      - cluster.initial_master_nodes=es_hot,es_warm,es_cold
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es_data_warm:/usr/share/elasticsearch/data
    networks:
      default:
        ipv4_address: 172.22.0.22
  elasticsearch3:
    image: elasticsearch
    restart: always
    container_name: es_cold
    environment:
      - cluster.name=es_cluster
      - node.name=es_cold
      - node.attr.box_type=cold
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.seed_hosts=es_hot,es_warm,es_cold
      - cluster.initial_master_nodes=es_hot,es_warm,es_cold
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es_data_cold:/usr/share/elasticsearch/data
    networks:
      default:
        ipv4_address: 172.22.0.23


volumes:
  es_data_hot:
    driver: local
  es_data_warm:
    driver: local
  es_data_cold:
    driver: local
      
networks:
  default:
    external:
      name: op_net
```

3. 启动 Elasticsearch 集群
```bash
docker-compose -f elasticsearch.yml up -d
```