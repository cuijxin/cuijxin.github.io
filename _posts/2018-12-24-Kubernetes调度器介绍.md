---
layout:     post
title:      Kubernetes调度器介绍
subtitle:   K8S
date:       2018-12-24
author:     
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - K8S
---

>原文链接：https://mp.weixin.qq.com/s/zXy5iYDTFxffzI7_IeLU7Q

## 前言

kube-scheduler 是 kubernetes 系统的核心组件之一，主要负责整个集群资源的调度功能，根据特定的调度算法和策略，将 Pod 调度到最有的工作节点上面去，从而更加合理、更加充分的利用集群的资源，这也是我们选择使用 kubernetes 一个非常重要的理由。如果一门新的技术不能帮助企业节约成本、提高效率，我相信是很难推进的。

## 调度流程

默认情况下，kube-scheduler 提供的默认调度器能够满足我们绝大多数的需求，我们前面和大家介绍的示例也基本上用的默认的策略，都可以保证我们的 Pod 可以被分配到资源充足的节点上运行。但是在实际的线上项目中，可能我们自己会比 kubernetes 更加了解我们自己的应用，比如我们希望一个 Pod 只能运行在特定的几个节点上，或者这几个节点只能用来运行特定类型的应用，这就需要我们的调度器能够可控。

kube-scheduler 是 kubernetes 的调度器，它的主要作用就是根据特定的调度算法和调度策略将 Pod 调度到合适的 Node 节点上去，是一个独立的二进制程序，启动之后会一直监听 API Server, 获取到 PodSpec.NodeName 为空的 Pod, 对每个 Pod 都会创建一个 binding.

![](/img/kube-scheduler.png)