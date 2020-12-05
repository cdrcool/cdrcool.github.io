---
title: 基于 Docker 搭建 Mysql 集群
date: 2020-02-07 03:04:00
categories: MySQL
tags:
    - MySQL
    - Docker
---
```bash
docker run -d `
  --network op_net `
  --hostname mysql `
  --name mysql `
  --restart=always `
  -e MYSQL_ROOT_PASSWORD=root `
  -p 3306:3306 `
  mysql `
  --lower_case_table_names=1
```