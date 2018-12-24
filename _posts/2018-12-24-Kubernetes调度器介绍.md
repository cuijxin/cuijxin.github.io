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

更详细的流程是这样的：

1. 首先，客户端通过 API Server 的 REST API 或者 kubectl 工具创建 Pod 资源
2. API Server 收到用户请求后，存储相关数据到 etcd 数据库中
3. 调度器监听 API Server 查看为调度（bind）的 Pod 列表，循环遍历地为每个 Pod 尝试分配节点，这个分配过程就是我们上面提到的两个阶段：
  A 预先阶段（Predicates），过滤节点，调度器用一组规则过滤掉不符合要求的 Node 节点，比如 Pod 设置了资源的 request，那么可用资源比 Pod 需要的资源少的主机显然就会被过滤掉
  B 优选阶段（Priorities），为节点的优先级打分，将上一阶段过滤出来的 Node 列表进行打分，调度器会考虑一些整体的优化策略，比如把 Deployment 控制的多个 Pod 副本分布到不同的主机上，使用最低负载的主机等等策略
4. 经过上面的阶段过滤后选择打分最高的 Node 节点和 Pod 进行 binding 操作，然后将结果存储到 etcd 中
5. 最后被选择出来的 Node 节点对应的 kubelet 去执行创建 Pod 的相关操作

其中 Predicates 过滤有一系列的算法可以使用，我们这里简单列举几个：

**1. PodFitsResources: 节点上剩余的资源是否大于 Pod 请求的资源**
**2. PodFitsHost: 如果 Pod 指定了 NodeName，检查节点名称是否和 NodeName 匹配**
**3. PodFitsHostPorts: 节点上已经使用的 port 是否和 Pod 申请的 port 冲突**
**4. PodSelectorMatches: 过滤掉和 Pod 指定的 label 不匹配的节点**
**5. NoDiskConflict：已经 mount 的 volume 和 Pod 指定的 volume 不冲突，除非它们都是只读的**
**6. CheckNodeDiskPressure: 检查节点磁盘空间是否符合要求**
**7. CheckNodeMemoryPressure: 检查节点内存是否够用**

除了这些过滤算法之外，还有一些其他的算法，更多更详细的我们可以查看源码文件：
k8s.io/kubernetes/pkg/scheduler/algorithm/predicates/predicates.go。

而 Priorities 优先级是由一系列键值对组成的，键是该优先级的名称，值是它的权重值，同样，我们这里给大家列举几个具有代表性的选项：
**1. LeastRequestedPriority: 通过计算 CPU 和内存的使用率来决定权重，使用率越低权重越高，当然正常肯定也是资源使用率越低权重越高，能给别的 Pod 运行的可能性就越大**
**2. SelectorSpredPriority: 为了更好的高可用，对同属于一个 Deployment 或者 RC 下面的多个 Pod 副本，尽量调度到多个不同的节点上，当一个 Pod 被调度的时候，会先去查找该 Pod 对应的 controller，然后查看该 controller 下面的已存在的 Pod，运行 Pod 越少的节点权重越高**
**3. ImageLocalityPriority: 就是如果在某个节点上已经有要使用的镜像了，镜像总大小值越大，权重就越高**
**4. NodeAffinityPriority：这个就是根据节点的亲和性来计算一个权重值，后面我们会详细讲解亲和性的使用方法**

