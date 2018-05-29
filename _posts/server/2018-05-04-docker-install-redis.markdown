---
layout: post
title:  "在 Docker 中安装 Redis, 然后使用 Spring 访问"
date:   2018-05-04 14:57:30 +0800
categories: server
---

* TOC
{:toc}


最近在做缓存有关的项目，在原有项目里写测试代码比较麻烦，先在本地搭个环境测试下。

## 在 Docker 中安装 Redis

```
# 在 DockerHub 上查找 redis 镜像
docker search redis

# 拉取最新的 redis 镜像
docker pull redis

# 生成并运行 redis 容器
# --name        : 指定容器名为 redis
# -p 6379:6379  : 把容器的 6379 端口映射到本机的 6379 端口上
# -d            : 分离模式，让容器在后台运行
docker container run --name redis -p 6379:6379 -d redis redis-server

# 杀死正在运行的 redis 容器
docker container kill redis

# 启动已经停止的 redis 容器
docker container restart redis
```

## 一个 Spring 连接 Redis 的简单例子

