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


# 资源对象

![]( {{site.url}}/asset/kubernetes-resource-object.png )

上图列出了 Kubernetes 当中比较重要的一些资源对象。核心的资源对象有两个: Pod 和 Service，这也反映了 Kubernetes 的两大重要功能：部署和服务发现。

Replica Set, Deployment, StatefulSet, DaemonSet, Job, Cron Job 这些资源对象，是一种描述性的配置，Master 节点中的 controller-manager 组件会根据这些配置来生成 Pod 资源，随后由 scheduler 组件根据 Pod 资源在 Node 上真正完成部署。

Service 表示由一组 Pod 提供的服务。Node 节点中的 kube-proxy 组件会根据 Service 信息来生成 ClusterIP -> Pod IP 的转发规则，这样无论在哪个 Node 都可以通过 Cluster IP 来访问 Service。

Ingress 是在 Service 上层的一个东西，它也是一份配置，定义了一些规则把 HTTP 流量转发到相应的 Service。为了让这份配置生效，需要部署一个 Ingress Controller 到某些 Pod 上。这个 Ingress Controller 当中应该包含一个代理软件，比如 Nginx，以及一个配置转换的脚本，这个脚本会根据 Ingress 配置生成 Nginx 规则并设置到 Nginx 上。然后流量先打到 Ingress Controller 的 Nginx 上，Nginx 根据配置把流量转发给合适的 Service。