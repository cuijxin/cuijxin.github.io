---
layout:     post
title:      Deployment概念详细解析
subtitle:   《Kubernetes指南》书摘
date:       2018-03-27
author:     cjx
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - K8S
---

> 译自 [《kubernetes官方文档》](https://github.com/kubernetes/kubernetes.github.io/blob/master/docs/concepts/workloads/controllers/deployment.md)

## Deployment是什么？

Deployment为Pod和Replica Set（下一代Replication Controller）提供声明式更新。

你只需要在Deployment中描述你想要的目标状态是什么，Deployment controller就会帮你将Pod和Replica Set的实际状态改变到你的目标状态。你可以定义一个全新的Deployment，也可以创建一个新的替换旧的Deployment。

一个典型的用例如下：

1. 使用Deployment来创建ReplicaSet。ReplicaSet在后台创建Pod。检查启动状态，看它是成功还是失败。

2. 然后，通过更新Deployment的PodTemplateSpec字段来声明Pod的新状态。这会创建一个新的ReplicaSet，Deployment会按照控制的速率将pod从旧的ReplicaSet移动到新的ReplicaSet中。

3. 如果当前状态不稳定，回滚到之前的Deployment revision。每次回滚都会更新Deployment的revision。

4. 扩容Deployment以满足更高的负载。

5. 暂停Deployment来应用PodTemplateSpec的多个修复，然后恢复上线。

6. 根据Deployment的状态判断上线是否hang住了。

7. 清除旧的不必要的ReplicaSet。

## 创建Deployment

下面是一个Deployment示例，它创建了一个Replica Set来启动3个nginx pod。

示例yaml文件内容如下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: nginx-deployment
spec:
    replicas: 3
    template:
        metadata:
            labels:
                app: nginx
        spec:
            containers:
                - name: nginx
                  image: nginx:1.7.9
                  ports:
                      - containerPort: 80
```

保存以上内容为nginx-deployment.yaml，命令行执行kubectl create命令：

```
$kubectl create -f nginx-deployment.yaml --record
deployment "nginx-deployment" created
```

将kubectl的 --record 的flag设置为 true 可以在annotation中记录当前命令创建或者升级了该资源。这在未来会很有用，例如，查看每个Deployment revision中执行了哪些命令。

然后立即执行 get 将获得如下结果：

```
$kubectl get deployments
NAME               DESIRED      CURRENT    UP-TO-DATE    AVAILABLE   AGE
nginx-deployment   3            0          0             0           1s                
```

输出结果表明我们希望的replica数是3（根据deployment中的 .spec.replicas 配置），当前replica数（ .status.replicas ）是0，最新的replica数（ .status.updatedReplicas ）是0，可用的replica数（ .status.availableReplicas ）是0。

过几秒后再执行 get 命令，将获得如下输出：

```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           5s
```

我们可以看到Deployment已经创建了3个replica，所有的replica都已经是最新的了（包含最新的pod template），可用的（根据Deployment中的 .spec.minReadySeconds 声明，处于已就绪状态的pod的最少个数）。执行 kubectl get rs 和 kubectl get pods 会显示Replica Set（RS）和Pod已创建。

```
$kubectl get rs
NAME                         DESIRED     CURRENT     READY    AGE
nginx-deployment-431080787   3           3           3        8m
```

你可能会注意到Replica Set的名字总是 {Deployment的名字}-{pod template的hash值}。