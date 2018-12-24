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

这个过程在我们看来好像比较简单，但在实际的生产环境中，需要考虑的问题就有很多了：
**1. 如何保证全部的节点调度的公平性？要知道并不是所有节点资源配置都是一样的。**
**2. 如何保证每个节点都能被分配资源？**
**3. 集群资源如何能够被高效利用？**
**4. 集群资源如何才能被最大化使用？**
**5. 如何保证 Pod 调度的性能和效率？**
**6. 用户是否可以根据自己的实际需求定制自己的调度策略？**

考虑到实际环境中的各种复杂情况，kubernetes 的调度器采用插件化的形式实现，可以方便用户进行定制或者二次开发，我们可以自定义一个调度器并以插件形式和 kubernetes 进行集成。

kubernetes 调度器的源码位于 kubernetes/pkg/scheduler 中，大体的代码目录结构如下所示：（不同版本目录结构可能不太一样）
```
kubernetes/pkg/scheduler
-- scheduler.go          // 调度相关的具体实现
|-- algorithm
|   |-- predicates       // 节点筛选策略
|   |-- priorities       // 节点打分策略
|-- algorithmprovider
|   |-- defaults         // 定义默认的调度器  
```
其中 Scheduler 创建和运行的核心程序，对应的代码在 pkg/schduler/scheduler.go，如果要查看 kube-scheduler 的入口程序，对应的代码在 cmd/kube-scheduler/scheduler.go。
调度主要分为以下几个部分：

**1. 首先是预选过程，过滤掉不满足条件的节点，这个过程称为 Predicates**
**2. 然后是优选过程，对通过的节点按照优先级排序，称之为 Priorities**
**3. 最后从中选择优先级最高的节点，如果中间任何一步骤有错误，就直接返回错误**

Predicates  阶段首先遍历全部节点，过滤掉不满足条件的节点，属于强制性规则，这一阶段输出的所有满足要求的 Node 将被记录并作为第二阶段的输入，如果所有的节点都不满足条件，那么 Pod 将会一直处于 Pending 状态，直到有节点满足条件，在这期间调度器会不断的重试。

所以我们在部署应用的时候，如果发现有 Pod 一直处于 Pending 状态，那么就是没有满足调度条件的节点，这个时候可以去检查下节点资源是否可用。

Priorities 阶段即再次对节点进行筛选，如果有多个节点都能满足条件的话，那么系统会按照节点的优先级（priorites）大小对节点进行排序，最后选择优先级最高的节点来部署 Pod 应用。

下面是调度过程的简单示意图：

![](/img/kube-scheduler1.png)