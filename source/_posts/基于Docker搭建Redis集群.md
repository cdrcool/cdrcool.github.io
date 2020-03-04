---
title: 基于 Docker 搭建 Redis 集群
date: 2020-02-17 17:45:00
categories: Redis
tags:
    - Redis
    - Docker
---
1. 创建 network
```bash
docker network create --driver bridge --subnet 172.22.0.0/16 --gateway 172.22.0.1  op_net
```

2. 创建 redis.yml
```yaml
version: '3.6'

services:
  redis-1:
    image: redis
    restart: always
    hostname: redis-1
    container_name: redis-1
    ports:
      - 6379:6379
    networks:
      default:
        ipv4_address: 172.22.0.26

  redis-2:
    image: redis
    restart: always
    hostname: redis-2
    container_name: redis-2
    ports:
      - 6380:6379
    command: redis-server --slaveof redis-1 6379
    networks:
      default:
        ipv4_address: 172.22.0.27

  redis-3:
    image: redis
    restart: always
    hostname: redis-3
    container_name: redis-3
    ports:
      - 6381:6379
    command: redis-server --slaveof redis-1 6379
    networks:
      default:
        ipv4_address: 172.22.0.28
      
networks:
  default:
    external:
      name: op_net
```

3. 启动 Redis 集群
```bash
docker-compose -f redis.yml up -d
```