---
layout:     post
title:      Kubernetes中的Persistent Volume解析
subtitle:   Kbernetes入门教程
date:       2018-07-04
author:     
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - K8S
---

本文档介绍了Kuberentes中PersistentVolume的当前状态。这是Kubernetes官方中文文档的初稿，欢迎在[官方PR反馈问题](https://github.com/kubernetes/kubernetes-docs-zh/pull/164)，建议在阅读本文档前先熟悉[volume](https://kubernetes.io/docs/concepts/storage/volumes/)。本文同时归档到[kubernetes-handbook](https://jimmysong.io/kubernetes-handbook)。

## 介绍

对于管理计算资源来说，管理存储资源明显是另一个问题。```PersistentVolume```子系统为用户和管理员提供了一个API，该API将如何提供存储的细节抽象了出来。为此，我们引入了两个新的API资源：```PersistentVolume```和```PersistentVolumeClaim```。

```PersistentVolume```（PV）是由管理员设置的存储，它是集群的一部分。就像节点是集群中的资源一样，PV也是集群中的资源。PV是Volume之类的卷插件，但具有独立于使用PV的Pod的生命周期。此API对象包含存储实现的细节，即NFS、iSCSI或特定于云供应商的存储系统。

```PersistentVolumeClaim```（PVC）是用户存储的请求。它与Pod相似。Pod消耗节点资源，PVC消耗PV资源。Pod可以请求特定级别的资源（CPU和内存）。PVC可以请求特定的大小和访问模式的存储资源（例如，可以以读/写一次或只读多次模式挂载）。

虽然```PersistentVolumeClaims```允许用户使用抽象存储资源，但用户需要具有不同性质（例如性能）的```PersistentVolume```来解决不同的问题。集群管理员需要能够提供各种各样的```PersistentVolume```，这些```PersistentVolume```的大小和访问模式可以各有不同，但不需要向用户公开实现这些卷的细节。对于这些需求，```StorageClass```资源可以实现。

请参阅[工作示例的详细过程](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)。

## PV和PVC的生命周期

PV属于集群中的资源。PVC是对这些资源的请求，也作为对资源的请求的检查。PV和PVC之间的相互作用遵循这样的生命周期：

### 配置（Provision）

有两种方式来配置PV：静态或动态。

#### 静态

集群管理员创建一些PV。它们带有可供集群用户使用的实际存储的细节。他们存在于Kubernetes API中，可用于消费。

#### 动态

当管理员创建的静态PV都不匹配用户的```PersistentVolumeClaim```时，集群可能会尝试动态地为PVC创建卷。此配置基于```StorageClass```：PVC必须请求[存储类](https://kubernetes.io/docs/concepts/storage/storage-classes/)，并且管理员必须创建并配置该类才能进行动态创建。声明该类为“”可以有效地禁用其动态配置。

要启用基于存储级别的动态存储配置，集群管理员需要启用API server上的```DefaultStorageClass```[准入控制器](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass)。例如，通过确保```DefaultStorageClass```位于API server组件的```--admission-control```标志，使用逗号分隔有序值列表中，可以完成此操作。有关API server命令行标志的更多歇息，请检查[kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)文档。

### 绑定

在动态配置的情况下，用户创建或已经创建了具有特定存储量的```PersistentVolumeClaim```以及某些访问模式。master中的控制环路监视新的PVC，寻找匹配的PV（如果可能），并将它们绑定在一起。如果为新的PVC动态调配PV，则该环路将始终将该PV绑定到PVC。否则，用户总会得到他们所请求的存储，但是容量可能超出要求的数量。一旦PV和PVC绑定后，```PersistentVolumeClaim```绑定是排他性的，不管它们是如何绑定的。PVC跟PV绑定是一对一的映射。

如果没有匹配的卷，声明将无限期地保持未绑定状态。随着匹配卷的可用，声明将被绑定。例如，配置了许多50Gi PV的集群将不会匹配请求100Gi的PVC。将100Gi的PV添加到集群时，可以绑定PVC。

### 使用

Pod使用PVC作为卷。集群检查PVC以查找绑定的卷并为集群挂载该卷。对于支持多种访问模式的卷，用户指定在使用PVC作为容器中的卷时所需的模式。

用户进行了声明，并且该声明是绑定的，则只要用户需要，绑定的PV就属于该用户。用户通过在Pod的volume配置中包含```persistentVolumeClaim```来调度Pod并访问用户声明的PV。请参阅下面的语法细节。
