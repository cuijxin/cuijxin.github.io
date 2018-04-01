---
layout:     post
title:      RC(Replica Set)的一些特性与作用
subtitle:   《Kubernetes指南》书摘
date:       2018-03-27
author:     cjx
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - K8S
---

## 引言

RC是Kubernetes系统中的核心概念之一，简单来说，它其实是定义了一个期望的场景，即声明某种Pod的副本数量在任意时刻都符合某个预期值，所以RC的定义包括如下几个部分。

1.Pod期待的副本数(replicas).
2.用于筛选目标Pod的Label Selector.
3.当Pod的副本数量小于预期数量的时候，用于创建新Pod的Pod模板(template)

......

### Replica Set

由于Replication Controller与Kubernetes代码中的模块Replication Controller同名，同时这个词也无法准确表达它的本意，所以在Kubernetes1.2的时候，它就升级成了另外一个新的概念---Replica Set，官方解释为“下一代的RC”，它与RC当前存在的唯一区别是：Replica Sets支持基于集合的Label selector (Set-based selector),而RC只支持基于等式的 Label Selector（equality-based selector），这使得Replica Set的功能更强。

### 发展趋势

kubectl命令行工具适用于RC的绝大部分命令都同样适用于Replica Set。此外，当前我们很少单独使用Replica Set，它主要被Deployment这个更高层的资源对象所使用，从而形成一整套Pod创建、删除、更新的编排机制。当我们使用Deployment时，无需关心它是如何创建和维护Replica Set的，这一切都是自动发生的。

Replica Set与Deployment这两个重要的资源对象逐步替换了之前的RC的作用，是Kubernetes1.3里Pod自动扩容（伸缩）这个关键功能实现的基础，也将继续在Kubernetes未来的版本中发挥重要的作用。

最后我们总结一下关于RC（Replica Set）的一些特性与作用。

1.在大多数情况下，我们通过定义一个RC实现Pod的创建过程及副本数量的自动控制。
2.RC里包括完整的Pod定义模板。
3.RC通过Label Selector机制实现对Pod副本的自动控制。
4.通过改变RC里的Pod副本数量，可以实现Pod的扩容或收缩功能。
5.通过改变RC里的Pod模板中的镜像版本，可以实现Pod的滚动升级功能。