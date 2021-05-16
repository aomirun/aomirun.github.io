---
title: "学习 servicecomb gochassis 第一天"
date: 2021-05-17T00:07:31+08:00
draft: false
toc: false
images:
tags: 
  - servicecomb
  - gochassis
  - go
  - micro
  - 微服务
---

## 学习 servicecomb gochassis 第一天

### 1. 本地安装 <服务中心 ServiceCenter>
我的学习笔计是基于Docker环境,使用Docker-compose来编排,各平台如何安装Docker和docker-compose可自行搜索学习.

创建 Docker 网络
```sh
$ docker network create -d bridge --subnet=172.27.0.0/16 --gateway=172.27.0.1 aomi
```
创建 docker-compose.yml 文件
```yml
version: "3"

services:
  service-center:
    image: servicecomb/service-center
    container_name: service-center
    ports:
      - "30100:30100"
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime
    networks:
      aomi:
        ipv4_address: 172.27.0.2
  
  service-frontend:
    image: servicecomb/scfrontend
    container_name: service-frontend
    environment:
      SC_ADDRESS: "service-center"
    ports:
      - "30103:30103"
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime
    networks:
      aomi:
        ipv4_address: 172.27.0.3

networks:
  aomi:
    external: true
```

启动环境
```sh
$ docker-compose up -d
```

访问本地Web端口 http://127.0.0.1:30103
见到如下图,即安装成功
![servicecomb_dashboard](/images/posts/servicecomb/servicecomb_dashboard.png "servicecomb_dashboard")
因之前做过测试,上面跑了两个测试服务,这个不影响.

### 创建项目
目录如下,项目可以去我的Github上面下载 https://github.com/aomirun/gochassis-start 

```sh
$ tree
.
├── Makefile
├── README.md
├── client
│   ├── Dockerfile
│   ├── conf
│   │   ├── chassis.yaml
│   │   └── microservice.yaml
│   └── main.go
├── docker-compose.yml
├── go.mod
├── go.sum
└── server
    ├── Dockerfile
    ├── conf
    │   ├── chassis.yaml
    │   └── microservice.yaml
    └── main.go

4 directories, 13 files

```
1. clone
```sh
$ git clone https://github.com/aomirun/gochassis-start
```

2. start
```sh
$ cd gochassis-start
$ make bd
$ docker-compose up -d
```

3. check
```sh
$ curl http://127.0.0.1:2002/hi
hello. go chassis
```
