---
layout:     post
title:      Kubernetes网络
subtitle:   《Kubernetes指南》书摘
date:       2018-2-24
author:     cjx
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - K8S
---

## Kubernetes网络

### Kubernetes网络模型

1. IP-per-Pod，每个Pod都拥有一个独立的IP地址，Pod内所有容器共享一个网络命名空间。

2. 集群内所有Pod都在一个直接连通的扁平网络中，可通过IP直接访问。
  
  1）所有容器之间无需NAT就可以直接互相访问。
  
  2）所有Node和所有容器之间无需NAT就可以直接互相访问。

  3）容器自己看到的IP跟其他容器看到的一样。

3. Service cluster IP尽可在集群内布访问，外部请求需要通过NodePort、LoadBalance或者Ingress来访问。

### 官方插件

1. kubenet:这是一个基于CNI bridge的网络插件（在bridge插件的基础上扩展了port mapping和traffic shaping），是目前推荐的默认插件。

2. CNI:CNI网络插件，需要用户将网络配置放到```/etc/cni/net.d```目录中，并将CNI插件的二进制文件放入```/opt/cni/bin```