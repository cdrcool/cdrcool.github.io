---
title: 基于 Docker 搭建 Mysql 集群
date: 2020-14-07 03:04:00
categories: Mysql
tags:
    - Mysql
    - Docker
---
```bash
docker run -d `
  --network op_net `
  --hostname mysql `
  --name mysql `
  --restart=always `
  -e MYSQL_ROOT_PASSWORD=admin123 `
  -p 3306:3306 `
  mysql
```