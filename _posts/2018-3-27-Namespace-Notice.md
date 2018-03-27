---
layout:     post
title:      Namespace注意事项
subtitle:   《Kubernetes指南》书摘
date:       2018-03-27
author:     cjx
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - K8S
---

## 引言

在 Kubernetes 中 Namespace 是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。

Namespace 常用来隔离不同的用户，比如 Kubernetes 自带的服务一般运行在名为 kube-system 的 Namespace 中。

### 注意事项：

### Namespace的状态

Namespace 包含两种状态“ Active ”和“ Terminating ”。在 Namespace 删除过程中，Namespace 状态被设置为“ Terminating ”。

### Namespace命名规则

Namespace 名称满足正则表达式 [a-z0-9]([-a-z0-9]*[a-z0-9])?，最大长度为63位。

### 删除Namespace

1. 删除一个Namespace会自动删除所有属于该Namespace的资源。

2. default 和 kube-system 命名空间不可删除。

3. PersistentVolumes 是不属于任何 Namespace 的，但 PersistentVolumeClaim 是属于某个特定 Namespace 的。

4. Events 是否属于 Namespace 取决于产生 events 的对象。

## 参考文档

[《Kubernetes Namespace》](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

[《Share a Cluster with Namespaces》](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/)