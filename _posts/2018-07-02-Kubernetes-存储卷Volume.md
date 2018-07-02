---
layout:     post
title:      Kubernetes-存储卷Volume
subtitle:   Kbernetes入门教程
date:       2018-07-02
author:     Daniel_Ji
header-img: img/post-bg-debug.png
catalog: true
tags:
    - K8S
---

## Kubernetes-存储卷Volume

### 存储卷概述

由于容器本身是非持久化的，因此需要解决在容器中运行应用程序遇到的一些问题。首先，当容器崩溃时，kubelet将重新启动容器，但是写入容器的文件将会丢失，容器将会以镜像的初始状态重新开始；第二，在通过一个Pod中一起运行的容器，通常需要共享容器之间一些文件。Kubernetes通过存储卷解决上述的两个问题。

在Docker有存储卷的概念，但Docker中存储卷只是磁盘的或另一个容器中的目录，并没有对其生命周期进行管理。Kubernetes的存储卷有自己的生命周期，它的生命周期与使用它的Pod的生命周期一致。因此，相比于在Pod中运行的容器来说，存储卷的存在时间会比其中的任何容器都长，并且在容器重新启动时会保留数据。当然，当Pod停止存在时，存储卷也将不再存在。Kubernetes支持多种类型的卷，而Pod可以同时使用各种类型和任意数量的存储卷。在Pod中通过指定下面的字段来使用存储卷：

1. ```spec.volumes:``` 通过此字段提供指定的存储卷；

2. ```spec.containers.volumeMounts:``` 通过此字段将存储卷挂载到容器中；

### 存储卷类型和示例

当前Kubernetes支持如下所列出的这些存储卷类型，并以hostPath、nfs和persistentVolumeClaim类型的存储卷为例，介绍如何定义存储卷，以及如何在Pod中被使用。

```
awsElasticBlockStore      flocker                  projected
azureDisk                 gcePersistentDisk        portworxVolume
azureFile                 gitRepo                  quobyte
cephfs                    glusterfs                rbd
configMap                 hostPath                 scaleIO
csi                       iscsi                    secret
downwardAPI               local                    storageos
emptyDir                  nfs                      vsphereVolume
fc(fibre channel)         persistentVolumeClaim
```

#### hostPath

hostPath类型的存储卷用于将宿主机的文件系统的文件或目录挂载到Pod中，除了需要指定path字段之外，在使用hostPath类型的存储卷时，也可以设置type，type支持的枚举值由下表所示。另外在使用hostPath时，需要注意下面的事项：

1. 具有相同配置的Pod（例如：从同一个podTemplate创建的），可能会由于被调度到不同的Node上而行为有所不同。

2. 在宿主机上创建的文件或目录，只有root用户具有写入的权限。你要么在容器中以root身份运行进程，要么在主机上修改文件或目录的权限，以便具备写入内容到hostPath存储卷中的能力。

![](/img/hostPath-01.png)