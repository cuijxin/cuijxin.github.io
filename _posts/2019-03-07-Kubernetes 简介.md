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

## Kubernetes主要概念

Kubernetes中的Node、Pod、Replication Controller、Service等都可以看作为资源对象，几乎所有的资源对象都可以通过Kubectl工具执行增删改查并将其保存在etcd中持久化存储。

Kubernetes通过对资源进行监控，并对比etcd库中保存的资源期望状态与当前环境中的资源实际状态的差异来实现对容器集群的自动控制与自动纠错。

### 1. Master

Master和Node都属于物理机或虚拟机，Master是集群的控制节点，负责集群的管理和控制，接收并执行Kubernetes的控制命令。

Master之上运行如下关键进程：

1. Kubernetes API Server（kube-apiserver）：提供标准的Http Rest接口，是Kubernetes资源增删改查的唯一入口，也是集群控制的入口进程；
2. Kubernetes Controller Manager（kube-controller-manager）：Kubernetes中所有资源对象的自动化控制中心，是资源对象的大总管；
3. Kubernetes Scheduler Manager（kube-scheduler-manager）：负责资源调度（pod调度）的进程；
4. etcd：Master节点上需要启动etcd服务，Kubernetes上所有资源对象的数据都会保存在etcd中。

### 2. Node

除了Master，Kubernetes集群中其他所有机器称之为Node节点。Node是Pod真正运行的主机，可以是物理机，也可以是虚拟机。Node节点可以在运行期间动态的增加到Kubernetes集群中，当Node被纳入集群之中后，kubelet会自动向Master注册自己从而被Master所管理，Master会分配给Node任务，当Node宕机时，其工作负载会被Master自动转移到其他Node上。

Node会定时向Master发送自身的信息，包括操作系统，Docker版本，机器CPU和内存使用情况，以及当前有哪些Pod运行在该Node上等。Master会根据这些信息获知每个Node的资源使用情况，从而实现高效均衡的资源调度策略。当某个Node超过指定时间没有上报信息时，会被Master判断为“失联”，该Node状态会被标记为“不可用（Not Ready）”，随后Master会触发”工作负载转移“的自动流程。

为了管理Pod，Node上运行的关键进程如下：

1. kubelet：负责Pod对应容器的创建、启停等任务，同时与Master节点密切协作，实现集群管理的基本功能；
2. kube-proxy：实现Kubernetes Service的通信与负载均衡机制的主要组件；
3. Docker Engine（docker）：Docker引擎，负责本机容器的创建及管理工作，管理容器。

![](/img/pod-0307.png)

不像其他的资源（如Pod和Namespace），Node本质上不是Kubernetes来创建的，Kubernetes只是管理Node上的资源。虽然可以通过Manifest创建一个Node对象，但Kubernetes也只是去检查是否真的是有这么一个Node，如果检查失败，也不会往Node上去调度Pod。这个检查是由Node Controller来完成的。Node Controller负责：

1. 维护Node状态。
2. 与Cloud Provider同步Node。
3. 给Node分配容器CIDR。
4. 删除带有NoExecute taint的Node上的Pods。

默认情况下，kubelet在启动时会向master注册自己，并创建Node资源。

每个Node都包括以下状态信息：

1. 地址：包括hostname、外网IP和内网IP。
2. 条件（Condition）：包括OutOfDisk、Ready、MemoryPressure和DiskPressure。
3. 容量（Capacity）：Node上的可用资源，包括CPU、内存和Pod总数。
4. 基本信息（Info）：包括内核版本、容器引擎版本、OS类型等。

Taints和tolerations属性用于保证Pod不被调度到不合适的Node上，Taint应用于Node上，而toleration则应用于Pod上（Toleration是可选的）。

### 3. Namespace

Namespace是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户。常见的pods、services、replication controllers和deployments等都是属于某一个Namespace的（默认是default），而node，persistentVolumes等则不属于任何Namespace。

Namespace常用来隔离不同的用户，比如Kuberentes自带的服务一般运行在名为kube-system的namespace中。

注意：

1. 删除一个namespace会自动删除所有属于该namespace的资源。
2. default和kube-system命名空间不可删除。
3. PersistentVolumes是不属于任何namespace的，但PersistentVolumeClaim是属于某个特定namespace的。
4. Events是否属于namespace取决于产生events的对象。

### 4. Label

Label是用户指定的键值对。可以附加到各种资源对象上，例如Node，Pod，Service，RC等。一个资源对象可以定义多个Lable，同一个Label可以应用到多个资源对象中。Label可在资源对象定义时指定，也能在运行过程中动态添加和删除。

通过Label Selector（标签选择器）可以筛选拥有某些标签的资源对象，相当于SQL中的WHERE条件，例如name=redis-server的Label Selector作用于Pod时，相当于条件WHERE name=redis-server。我们可以通过为指定的资源对象添加多个Label实现多维度的资源分组管理。

### 5. Pod

Pod是一组紧密关联的容器集合，它们共享IPC、Network和UTC namespace，是Kubernetes调度的基本单位。Pod的设计理念是支持多个容器在一个Pod中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。

![](/img/pod-0307-01.png)

Pod的类别：

1. 静态Pod（Static Pod）：不存放在etcd存储中，而是存放在某个具体的Node上的一个具体文件中并且只在该Node上启动运行；
2. 普通Pod：普通Pod一旦被创建，会被立即放入etcd存储，随后被Kubernetes Master调度到某个具体的Node上并绑定（Binding），随后该Pod被对应的Node上的Kubelet进程实例化成一组相关的docker容器并启动起来。

容器类别：

1. Pause容器：根容器，其IP和挂载的Volume被业务容器所共享；
2. 业务容器：用户自定义的容器。

Pod的特征：

1. 包含多个共享IPC、Network和UTC namespace的容器，可直接通过localhost通信。
2. 所有Pod内容器都可以访问共享的Volume，可以访问共享数据。
3. Pod一旦调度后就跟Node绑定，即使Node挂掉也不会重新调度，推荐使用Deployment、Daemonsets等控制器来容错。
4. 优雅终止：Pod删除的时候先给其内的进程发送SIGTERM，等待一段时间（grace period）后才强制停止依然还在运行的进程。
5. 特权容器（通过SecurrityContext配置）具有改变系统配置的权限（在网络插件中大量使用）。

在默认情况下，Pod里的某个容器停止运行时，Kubernetes会自动检查到这个问题并且重新启动这个Pod（重启Pod里的所有容器），如果Pod所在的Node宕机，则会将这个Node上的所有Pod重新调度到其他节点。

### 6. Replication Controller

Replication Controller（简称RC）用于定义期望值，例如声明某种Pod的副本数量在任意时刻都符合某个预期值，所以其定义包括如下几个部分：

1. Pod期待的副本数；
2. 用于筛选目标Pod的Label Selector；
3. 当Pod的副本小于预期数量的时候，用于创建新Pod的Pod模版。

当RC被定义并提交到Kubernetes集群中，Master节点的Controller Manager组件就会得到通知，从而依据RC的定义，定期巡检系统中存活的目标Pod，确保目标Pod实例的数量刚好等于此RC的期望值，如果有过多的Pod副本在运行，系统就会停掉一些Pod，否则自动创建Pod。通过RC，Kubernetes实现了用户应用集群的高可用性。

在运行时，能够通过动态更改RC的副本数量，来实现Pod的动态缩放。

删除RC时，不会影响通过该RC已创建好的Pod，如果想要删除其创建的Pod，可以通过设置其副本数量为0实现。另外，Kubectl提供了stop和delete命令来一次性删除RC及其控制的全部Pod。

此外，RC可以实现滚动升级，蓝绿部署。

总结RC特性及作用：

1. 在大多数情况下，我们通过定义一个RC实现Pod的创建过程以及副本数量的自动控制；
2. RC里包括完整的Pod定义模版；
3. RC通过Label Selector机制实现对Pod副本的自动控制；
4. 通过改变RC的Pod副本数量，可以实现Pod的扩容和缩容功能；
5. 通过改变RC里Pod模版中的镜像版本，可以实现Pod的滚动升级功能；
