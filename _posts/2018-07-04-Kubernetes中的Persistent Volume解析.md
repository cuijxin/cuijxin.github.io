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