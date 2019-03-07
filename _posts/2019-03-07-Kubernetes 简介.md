---
layout:     post
title:      Kubernetes简介
subtitle:   安全
date:       2019-03-07
author:     
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - K8S
---

## Kubernetes是什么？

Kubernetes基本上是这两年最热门、最被人熟知的技术了，它为软件工程师提供了强大的容器编排能力，模糊了开发和运维之间的边界，让我们开发、管理和维护一个大型的分布式系统和项目变得更加容易。

作为一个目前在生产环境已经广泛使用的开源项目，Kubernetes被定义成一个用于自动化部署、扩容和管理容器应用的开源系统；它将一个分布式软件的一组容器打包成一个个更容易管理和发现的逻辑单元。

在Kubernetes统治了容器编排这一领域之前，其实也有很多容器编排方案，例如compose和Swarm，但是在运维大规模、复杂的集群时，这些方案基本已经都被Kubernetes替代了。

Kubernetes将已经打包好的应用镜像进行编排，所以如果没有容器技术的发展和微服务框架中复杂的应用关系，其实也很难找到合适的应用场景去使用，所以Kubernetes有两大核心“依赖”---容器技术和微服务架构。

### 容器技术

Docker已经是容器技术的事实标准了，它让开发者将自己的应用以及依赖打包到一个可移植的容器中，让应用程序的运行可以实现环境无关。

我们能通过Docker实现进程、网络以及挂载点和文件系统隔离的环境，并且能够对宿主机资源进行分配，这能够让我们在同一个机器上运行多个不同的Docker容器，任意一个Docker的进程都不需要关心宿主机的依赖，都各自在镜像构建时完成依赖的安装和编译等工作，这也是为什么Docker是Kubernetes项目的一个重要依赖。

### 微服务架构

如果今天的软件并不是特别复杂并且需要承载的峰值流量不是特别多，那么后端项目的部署其实也只需要在虚拟机上安装一些简单的依赖，将需要部署的项目编译后运行就可以了。

但是随着软件变得越来越复杂，一个完整的后端服务不再是单体服务，而是由多个职责和功能不同的服务组成，服务之间复杂的拓扑关系以及单机已经无法满足的性能需求使得软件的部署和运维工作变得非常复杂，这也就使得部署和运维大型集群变成了非常迫切的需求。

### 小结

Kubernetes的出现不仅主宰了容器编排的市场，更改变了过去的运维方式，不仅将开发与运维之间边界变得更加模糊，而且让DevOps这一角色变得更加清晰，每一个软件工程师都可以通过Kubernetes来定义服务之间的拓扑关系、线上的节点个数、资源使用量并且能够快速实现水平扩容、蓝绿部署等在过去复杂的运维操作。

## Kubernetes核心组件

### 1.核心组件

![](/img/hxzj.png)

Kubernetes主要由以下几个核心组件组成：

1. etcd保存了整个集群的状态；
2. apiservier提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
3. controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
4. scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
5. kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
6. Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
7. kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；

### 2.组件通信

Kubernetes多组件之间的通信原理为

1. apiserver负责etcd存储的所有操作，且只有apiserver才直接操作etcd集群；
2. apiserver对内（集群中的其他组件）和对外（用户）提供统一的REST API，其他组件均通过apiserver进行通信；
        a. controller manager、scheduler、kube-proxy和kubelet等均通过apiserver watch API检测资源变换情况，并对资源作相应的操作
        b. 所有需要更新资源状态的操作均通过apiserver的REST API进行
3. apiserver也会直接调用kubelet API（如logs，exec，attach等），默认不校验kubelet证书，但可以通过--kubelet-certificate-authority参数开启（而GKE通过SSH隧道保护它们之间的通信）

比如典型的创建Pod 的流程为：

![](/img/create_pod.png)

1. 用户通过REST API创建一个Pod；
2. apiserver将其写入etcd；
3. scheduler检测到未绑定Node的Pod，开始调度并更新Pod的Node绑定；
4. kubelet检测到有新的Pod调度过来，通过container runtime运行该Pod；
5. kubelet通过container runtime取到Pod状态，并更新到apiserver中。

### 3.端口号

![](/img/port.png)

Master node(s)

![](/img/master_node.png)

Workder node(s)

![](/img/work_node.png)