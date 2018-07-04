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