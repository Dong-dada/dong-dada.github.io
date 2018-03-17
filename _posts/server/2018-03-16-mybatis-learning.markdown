---
layout: post
title:  "MyBatis 基础知识"
date:   2018-03-16 15:54:30 +0800
categories: server
---

* TOC
{:toc}


## 简介

MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生信息，将接口和 Java 的 POJOs(Plain Old Java Objects,普通的 Java对象)映射成数据库中的记录。

无论是用过的 hibernate,Mybatis, 你都可以发现他们有一个共同点：
1. 从配置文件(通常是 XML 配置文件中)得到 sessionfactory.
2. 由 sessionfactory 产生 session
3. 在 session 中完成对数据的增删改查和事务提交等.
4. 在用完之后关闭 session 。
5. 在 Java 对象和 数据库之间有做 mapping 的配置文件，也通常是 xml 文件





