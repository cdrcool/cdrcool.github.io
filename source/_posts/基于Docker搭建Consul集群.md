---
title: 基于 Docker 搭建 Consul 集群
date: 2020-02-29 15:15:00
categories: Consul
tags:
    - Consul
    - Docker
---
1. 创建 network
```bash
docker network create --driver bridge --subnet 172.22.0.0/16 --gateway 172.22.0.1  op_net
```

2. 创建 consul.yml
```yaml
version: '3.6'

services:
  consul-1:
    image: consul
    hostname: consul-1
    container_name: consul-1
    command: agent -server -bootstrap-expect=3 -node=consul-1 -bind=0.0.0.0 -client=0.0.0.0 -datacenter=dc1
    networks:
      default:
        ipv4_address: 172.22.0.31

  consul-2:
    image: consul
    hostname: consul-2
    container_name: consul-2
    command: agent -server -retry-join=consul-1 -node=consul-2 -bind=0.0.0.0 -client=0.0.0.0 -datacenter=dc1
    depends_on:
        - consul-1
    networks:
      default:
        ipv4_address: 172.22.0.32

  consul-3:
    image: consul
    hostname: consul-3
    container_name: consul-3
    command: agent -server -retry-join=consul-1 -node=consul-3 -bind=0.0.0.0 -client=0.0.0.0 -datacenter=dc1
    depends_on:
        - consul-1
    networks:
      default:
        ipv4_address: 172.22.0.33

  consul-4:
    image: consul
    hostname: consul-4
    container_name: consul-4
    ports:
      - 8500:8500
    command: agent -retry-join=consul-1 -node=consul-4 -bind=0.0.0.0 -client=0.0.0.0 -datacenter=dc1 -ui
    depends_on:
        - consul-2
        - consul-3
    networks:
      default:
        ipv4_address: 172.22.0.34

networks:
  default:
    external:
      name: op_net
```

3. 启动 Consul 集群
```bash
docker-compose -f consul.yml up -d
```

4. 访问 localhost:8500/