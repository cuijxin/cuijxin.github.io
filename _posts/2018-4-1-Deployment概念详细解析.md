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
$ kubectl create -f nginx-deployment.yaml --record
deployment "nginx-deployment" created
```

将kubectl的 --record 的flag设置为 true 可以在annotation中记录当前命令创建或者升级了该资源。这在未来会很有用，例如，查看每个Deployment revision中执行了哪些命令。

然后立即执行 get 将获得如下结果：
```
$ kubectl get deployments
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
$ kubectl get rs
NAME                         DESIRED     CURRENT     READY    AGE
nginx-deployment-431080787   3           3           3        8m
```

你可能会注意到Replica Set的名字总是 {Deployment的名字}-{pod template的hash值}。
```
$ kubectl get pods --show-labels
nginx-deployment-431080787-sb2pb   1/1       Running   0          15m       app=nginx,pod-template-hash=431080787
nginx-deployment-431080787-v3p0j   1/1       Running   0          15m       app=nginx,pod-template-hash=431080787
nginx-deployment-431080787-v5t7h   1/1       Running   0          15m       app=nginx,pod-template-hash=431080787
```

刚创建的Replica Set将保证总是有3个nginx的pod存在。

注意：你必须在Deployment中的selector指定正确pod template label（在该示例中是app = nginx），不要跟其他的controller搞混了（包括Deployment、Replica Set、Replication Controller等）。Kubernetes本身不会阻止你这么做，如果你真的这么做了，这些controller之间会相互打架，并可能导致不正确的行为。

## 更新Deployment

注意：Deployment的rollout当且仅当Deployment的pod template（例如 .spec.template）中的label更新或者镜像更改时被触发。其他更新，例如扩容Deployment不会触发rollout。

假如我们现在想要让nginx pod使用 nginx：1.9.1的镜像来代替原来的nginx：1.7.9的镜像。
```
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
deployment "nginx-deployment" image updated
```

我们也可以使用edit命令来编辑Deployment，修改 .spec.template.spec.containers[0].image，将 nginx:1.7.9 改写nginx:1.9.1。
```
$ kubectl edit deployment/nginx-deployment
deployment "nginx-deployment" edited
```

查看rollout的状态，只要执行：
```
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```

Rollout成功后，get Deployment：
```
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           40m
```

UP-TO-DATE的replica的数目已经达到了配置中要求的数据。

CURRENT的replica数表示Deployment管理的replica数量，AVAILABLE的replica数是当前可用的replica数量。

我们通过执行kubectl get rs 可以看到Deployment更新了Pod，通过创建一个新的Replica Set并扩容了3个replica，同时将原来的Replica Set所容到了0个replica。
```
$ kubectl get rs
nginx-deployment-2078889897   3         3         3         21m
nginx-deployment-431080787    0         0         0         1h
```

执行 get pods 只会看到当前的新的pod：
```
$ kubectl get pods
nginx-deployment-2078889897-fqwgd   1/1       Running   0          22m
nginx-deployment-2078889897-n4zvj   1/1       Running   0          22m
nginx-deployment-2078889897-vh0rs   1/1       Running   0          22m
```

下次更新这些pod的时候，只需要更新Deployment中的pod的template即可。

Deployment可以保证在升级时只有一定数量的Pod是down的。默认的，它会确保至少有比期望的Pod数量少一个的Pod是up状态的（最多一个不可用）。

Deployment同时也可以确保只创建出超过期望数量的一定数量的Pod。默认的，它会确保最多比期望的Pod数量多一个的Pod是up的（最多1个surge）。

在未来的Kubernetes版本中，将从1-1变成25%-25%。

例如，如果你自己看下上面的Deployment，你会发现，开始创建一个新的Pod，然后删除一些旧的Pod再创建一个新的。当新的Pod创建出来之前不会杀掉旧的Pod。这样能够确保可用的Pod数量至少有2个，Pod的总数最多4个。
```
$ kubectl describe deployments
Name:			nginx-deployment
Namespace:		default
CreationTimestamp:	Tue, 03 Apr 2018 00:15:55 +0800
Labels:			app=nginx
Annotations:		deployment.kubernetes.io/revision=2
Selector:		app=nginx
Replicas:		3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:		RollingUpdate
MinReadySeconds:	0
RollingUpdateStrategy:	1 max unavailable, 1 max surge
Pod Template:
  Labels:	app=nginx
  Containers:
   nginx:
    Image:		nginx:1.9.1
    Port:		80/TCP
    Environment:	<none>
    Mounts:		<none>
  Volumes:		<none>
Conditions:
  Type		Status	Reason
  ----		------	------
  Available 	True	MinimumReplicasAvailable
OldReplicaSets:	<none>
NewReplicaSet:	nginx-deployment-2078889897 (3/3 replicas created)
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason			Message
  ---------	--------	-----	----			-------------	--------	------			-------
  1m		1m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-431080787 to 3
  9s		9s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2078889897 to 1
  9s		9s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-431080787 to 2
  9s		9s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2078889897 to 2
  7s		7s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-431080787 to 1
  7s		7s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2078889897 to 3
  6s		6s		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-431080787 to 0

```
我们可以看到当我们刚开始创建这个Deployment的时候，创建了一个Replica Set（nginx-deployment-431080787），并直接扩容到了3个replica。当我们更新这个Deployment的时候，它会创建一个新的Replica Set（nginx-deployment-2078889897），将它扩容到1个replica，然后所容原先的Replica Set到两个replica，此时满足至少两个Pod是可用状态，同一时刻最多有4个Pod处于创建的状态。

接着继续使用相同的rolling update策略扩容新的Replica Set和所容旧的Replica Set。最终，将会在新的Replica Set中有3个可用的replica，旧的Replica Set的replica数目变成0。

## Rollover（多个rollout并行）

每当Deployment controller观测到有新的deployment被创建时，如果没有已存在的Replica Set来创建期望个数的Pod的话，就会创建出一个新的Replica Set来做这件事。已存在的Replica Set控制label匹配 .spec.selector 但是template跟 .spec.template 不匹配的Pod缩容。最终，新的Replica Set将会扩容出 .spec.replicas 指定数目的Pod，旧的Replica Set会缩容到0。

如果你更新了一个已存在并正在进行中的Deployment，每次更新Deployment都会创建一个新的Replica Set并扩容它，同时回滚之前扩容的Replica Set---将它添加到旧的Replica Set列表，开始缩容。

例如，假如你创建了一个有5个 nginx:1.7.9 replica的Deployment，但是当还只有3个 nginx:1.7.9 的replica创建出来的时候你就开始更新含有5个 nginx:1.9.1 replica的Deployment。在这种情况下，Deployment会立即杀掉已创建的3个 nginx:1.7.9 的Pod，并开始创建 nginx:1.9.1 的Pod。它不会等到所有的5个 nginx:1.7.9 的Pod都创建完成后才开始执行滚动更新。

## 回退Deployment

有时候你可能想回退一个Deployment，例如，当Deployment不稳定时，比如一直crash looping。

默认情况下，kubernetes会在系统中保存前两次的Deployment的rollout历史记录，以便你可以随时回退（你可以修改 revision history limit 来更改保存的revision数）。

注意：只要Deployment的rollout被触发就会创建一个revision。也就是说当且仅当Deployment的Pod template（如 .spec.template ）被更改，例如更新template中的label和容器镜像时，才会创建出一个新的revision。

其他的更新，比如扩容Deployment不会创建revision---因此我们可以很方便的手动或者自动扩容。这意味着当你回退到历史revision时，只有Deployment中的Pod template部分才会回退。

假设我们在更新Deployment的时候犯了一个拼写错误，将镜像的名字写成了 nginx:1.91，而正确的名字应该是 nginx:1.9.1 :
```
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment "nginx-deployment" image updated
```
Rollout将会卡住。
```
$ kubectl rollout status deployments nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
```
按住Ctrl-C停止上面的rollout状态监控。

你会看到旧的replicas（nginx-deployment-2078889897和nginx-deployment-431080787）和新的replicas（nginx-deployment-2558903419）数目都是两个。
```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-2078889897   2         2         2         32m
nginx-deployment-2558903419   2         2         0         39s
nginx-deployment-431080787    0         0         0         33m
```
看下创建Pod，你会看到有两个新的Replica Set创建的Pod处于ImagePullBackOff状态，循环拉取镜像。
```
$ kubectl get pods
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-2078889897-4jkvp   1/1       Running            0          34m
nginx-deployment-2078889897-lwkcm   1/1       Running            0          34m
nginx-deployment-2558903419-k12j9   0/1       ImagePullBackOff   0          2m
nginx-deployment-2558903419-sbfbk   0/1       ImagePullBackOff   0          2m
```

注意，Deployment controller会自动停止坏的rollout，并停止扩容新的Replica Set。
```
$ kubectl describe deployment
Name:			nginx-deployment
Namespace:		default
CreationTimestamp:	Tue, 03 Apr 2018 00:15:55 +0800
Labels:			app=nginx
Annotations:		deployment.kubernetes.io/revision=3
Selector:		app=nginx
Replicas:		3 desired | 2 updated | 4 total | 2 available | 2 unavailable
StrategyType:		RollingUpdate
MinReadySeconds:	0
RollingUpdateStrategy:	1 max unavailable, 1 max surge
Pod Template:
  Labels:	app=nginx
  Containers:
   nginx:
    Image:		nginx:1.91
    Port:		80/TCP
    Environment:	<none>
    Mounts:		<none>
  Volumes:		<none>
Conditions:
  Type		Status	Reason
  ----		------	------
  Available 	True	MinimumReplicasAvailable
OldReplicaSets:	nginx-deployment-2078889897 (2/2 replicas created)
NewReplicaSet:	nginx-deployment-2558903419 (2/2 replicas created)
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason			Message
  ---------	--------	-----	----			-------------	--------	------			-------
  42m		42m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-431080787 to 3
  41m		41m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2078889897 to 1
  41m		41m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-431080787 to 2
  41m		41m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2078889897 to 2
  41m		41m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-431080787 to 1
  41m		41m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2078889897 to 3
  41m		41m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-431080787 to 0
  9m		9m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2558903419 to 1
  9m		9m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-2078889897 to 2
  9m		9m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2558903419 to 2
```
为了修复这个问题，我们需要回退到稳定的Deployment revision。

### 检查Deployment升级的历史记录

首先，检查Deployment的revision：

```
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION	CHANGE-CAUSE
1		kubectl create --filename=nginx-deployment.yaml --record=true
2		kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
3		kubectl set image deployment/nginx-deployment nginx=nginx:1.91
```
因为我们创建Deployment的时候使用了--record 参数可以记录命令，我们可以很方便的查看每次revision的变化。

查看单个revision的详细信息：
```
$ kubectl rollout history deployment/nginx-deployment --revision=2
deployments "nginx-deployment" with revision #2
Pod Template:
  Labels:	app=nginx
	pod-template-hash=2078889897
  Annotations:	kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
  Containers:
   nginx:
    Image:	nginx:1.9.1
    Port:	80/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

### 回退到历史版本

现在，我们可以决定回退当前的rollout到之前的版本：
```
$ kubectl rollout undo deployment/nginx-deployment
deployment "nginx-deployment" rolled back
```

也可以使用 --to-revision 参数指定某个历史版本：
```
$kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment "nginx-deployment" rolled back
```

与rollout相关的命令详细文档见kubectl rollout。

该Deployment现在已经回退到了先前的稳定版本。如你所见，Deployment controller产生了一个回退到revision 2的 DeploymentRollback的event。

```
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         4         2            2           14m
$ kubectl describe deployment
Name:			nginx-deployment
Namespace:		default
CreationTimestamp:	Tue, 03 Apr 2018 01:03:14 +0800
Labels:			app=nginx
Annotations:		deployment.kubernetes.io/revision=3
			kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.91
Selector:		app=nginx
Replicas:		3 desired | 2 updated | 4 total | 2 available | 2 unavailable
StrategyType:		RollingUpdate
MinReadySeconds:	0
RollingUpdateStrategy:	1 max unavailable, 1 max surge
Pod Template:
  Labels:	app=nginx
  Containers:
   nginx:
    Image:		nginx:1.91
    Port:		80/TCP
    Environment:	<none>
    Mounts:		<none>
  Volumes:		<none>
Conditions:
  Type		Status	Reason
  ----		------	------
  Available 	True	MinimumReplicasAvailable
OldReplicaSets:	nginx-deployment-2078889897 (2/2 replicas created)
NewReplicaSet:	nginx-deployment-2558903419 (2/2 replicas created)
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason			Message
  ---------	--------	-----	----			-------------	--------	------			-------
  14m		14m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-431080787 to 3
  13m		13m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2078889897 to 1
  13m		13m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-431080787 to 2
  13m		13m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2078889897 to 2
  13m		13m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-431080787 to 1
  13m		13m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2078889897 to 3
  13m		13m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-431080787 to 0
  11m		11m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled up replica set nginx-deployment-2558903419 to 1
  11m		11m		1	deployment-controller			Normal		ScalingReplicaSet	Scaled down replica set nginx-deployment-2078889897 to 2
  11m		11m		1	deployment-controller			Normal		ScalingReplicaSet	(combined from similar events): Scaled up replica set nginx-deployment-2558903419 to 2
```

### 清理Policy 

你可以通过设置 .spec.revisionHistoryLimit 项来指定deployment最多保留多少revision历史记录。默认的会保留所有的revision；如果将该项设置为0，Deployment就不允许回退了。

## Deployment扩容

你可以使用以下命令扩容Deployment:

```
$ kubectl scale deployment nginx-deployment --replicas 10
deployment "nginx-deployment" scaled
```

假设你的集群中启用了horizontal pod autoscaling，你可以给Deployment设置一个autoscaler，基于当前Pod的CPU利用率选择最少和最多的Pod数。

```
$ kubectl autoscale deployment nginx-deployment --min=10 --max=15 --cpu-percent=80
deployment "nginx-deployment" autoscaled
```

### 比例扩容

RollingUpdate Deployment支持同时运行一个应用的多个版本。当你或者autoscaler扩容一个正在rollout中（进行中或者已经暂停）的RollingUpdate Deployment的时候，为了降低风险，Deployment controller将会平衡已存在的active的ReplicaSets（有Pod的ReplicaSets）和新加入的replicas。这被称为比例扩容。

例如，你正在运行中含有10个replica的Deployment。maxSurge=3，maxUnavailable=2。

```
$ kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   10        10        10           10          11h
```

你更新了一个镜像，而在集群内部无法解析。

```
$ kubectl set image deploy/nginx-deployment nginx=nginx:sometag
deployment "nginx-deployment" image updated
```

镜像更新启动了一个包含ReplicaSet nginx-deployment-418298827的新的rollout，但是它被阻塞了，因为我们上面提到的maxUnavailable。

因为我的机器和书上说的显示的效果不一致，这里就不摘录，等我搞清楚其中的原因再说吧。
……



