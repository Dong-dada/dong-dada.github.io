---
layout: post
title:  "Kubernetes - minikube"
date:   2020-10-27 16:11:30 +0800
categories: k8s
---

* TOC
{:toc}

# 简介

Minikube 是一个工具，可以在你的机器上部署出 kubernetes 环境。其架构如下图：

![]({{ site.url }}/asset/k8s-minikube-arch.jpeg)

简单来说就是在机器上创建一个虚拟机，虚拟机里部署了 k8s 的 master 组件，由此构建出一个 k8s 的 master 节点。

