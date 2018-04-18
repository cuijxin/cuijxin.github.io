---
layout:     post
title:      StatefulSet概念及注意事项
subtitle:   《Kubernetes指南》书摘
date:       2018-04-18
author:     cjx
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - K8S
---

## StatefulSet

StatefulSet是为了解决有状态服务的问题（对应Deployments和ReplicaSets是为无状态服务而设计），其应用场景包括：

1. 稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现。

2. 稳定的网络标志，即Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）来实现。

3. 有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依序进行（即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态），基于init containers来实现。

4. 有序收缩，有序删除（即从N-1到0）。

从上面的应用场景可以发现，StatefulSet由以下几个部件组成：

1. 用于定义网络标志（DNS domain）的Headless Service。

2. 用于创建PersistentVolumes的volumeClaimTemplates。

3. 定义具体应用的StatefulSet。

## StatefulSet注意事项

1. 还在beta状态，需要kubernetes v1.5版本以上才支持。

2. 所有Pod的Volume必须使用PersistentVolume或者是管理员事先创建好。

3. 为了保证数据安全，删除StatefulSet时不会删除Volume。

4. StatefulSet需要一个Headless Service来定义DNS domain，需要在StatefulSet之前创建好。
