---
layout:     post
title:      DaemonSet概念及注意事项
subtitle:   《Kubernetes指南》书摘
date:       2018-04-18
author:     cjx
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - K8S
---

## DaemonSet

DaemonSet保证在每个Node上都运行一个容器副本，常用来部署一些集群的日志、监控或者其他系统管理应用。典型的应用包括：

1. 日志收集，比如fluentd，logstash等。

2. 系统监控，比如Prometheus Node Exporter，collectd，New Relic agent，Ganglia gmond等。

3. 系统程序，比如kube-proxy，kube-dns，glusterd，ceph等。

### 滚动更新

v1.6+ 支持DaemonSet的滚动更新，可以通过 ```.spec.updateStrategy.type``` 设置更新策略。目前支持两种策略：

1. OnDelete：默认策略，更新模板后，只有手动删除了旧的Pod后才会创建新的Pod。

2. RollingUpdate：更新DaemonSet模板后，自动删除旧的Pod并创建新的Pod。

在使用RollingUpdate策略时，还可以设置

1. ```.spec.updateStrategy.rollingUpdate.maxUnavailable```，默认1。

2. ```spec.minReadySeconds```，默认0。

### 回滚

v1.7+还支持回滚

```
# 查询历史版本
$ kubectl rollout history daemonset <daemonset-name>

# 查询某个历史版本的详细信息
$ kubectl rollout history daemonset <daemonset-name> --revision=1

# 回滚
$ kubectl rollout undo daemonset <daemonset-name> --to-revision=<revision>

# 查询回滚状态
$ kubectl rollout status ds/<daemonset-name>
```

### 指定Node节点

DaemonSet会忽略Node的unschedulable状态，有两种方式来指定Pod只运行在指定的Node节点上：

1. nodeSelector：只调度到匹配指定label的Node上。

2. nodeAffinity；功能更丰富的Node选择器，比如支持集合操作。

3. podAffinity：调度到满足条件的Pod所在的Node上。

#### nodeSelector示例

首先给Node打上标签

```
$ kubectl label nodes node-01 disktype=ssd
```

然后在daemonset中指定nodeSelector为 disktype=ssd :

```
spec：
  nodeSelector:
    disktype: ssd
```


