---
layout: post
title:  "Kubernetes - 基础知识"
date:   2021-03-01 13:35:30 +0800
categories: k8s
---

* TOC
{:toc}


# 基本组件

![]( {{site.url}}/asset/k8s-basic-components.svg )

在一个 Kubernetes 集群当中，有许多机器(虚拟机、物理机等)，这些机器被划分为两类：
- Master
  - 也就是这个集群的管理中心，一般为了高可用会至少部署三台机器；
  - Master 上需要部署 apiserver, controller-manager, scheduler 三个关键进程；
- Node
  - 工作节点，也就是实际运行工作负载的机器
  - Node 上需要部署 kubelet, kube-proxy, Docker-Engine 三个关键进程；

总的来说，Kubernetes 就是通过一堆抽象的资源对象来管理机器的系统，主要是解决怎么把程序部署到机器上这件事。资源对象这种抽象的概念被保存在 etcd 存储当中，通过 master 节点中的 API Server 进程可以访问这些资源对象。
