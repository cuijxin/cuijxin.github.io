---
layout:     post
title:      Kubernetes存储卷
subtitle:   《Kubernetes指南》书摘
date:       2018-04-07
author:     cjx
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - K8S
---

## Kubernetes存储卷

我们知道默认情况下容器的数据都是非持久化的，在容器消亡以后数据也跟着丢失，所以Docker提供了Volume机制以便将数据持久化存储。类似的，Kubernetes提供了更强大的Volume机制和丰富的插件，解决了容器数据持久化和容器间共享数据的问题。

与Docker不同，Kubernetes Volume的生命周期与Pod绑定。

1. 容器挂掉之后，Kubelet再次重启容器时，Volume的数据依然还在
2. 而Pod删除时，Volume才会清理。数据是否丢失取决于具体的Volume类型，比如emptyDir的数据会丢失，而PV的数据则不会丢失。

### Volume类型

目前，Kubernetes支持以下Volume类型：

1. emptyDir
2. hostPath
3. gcePersistentDisk
4. awsElasticBlockStore
5. nfs
6. iscsi
7. flocker
8. glusterfs
9. rbd
10. cephfs
11. gitRepo
12. secret
13. persistentVolumeClaim
14. downwardAPI
15. azureFileVolume
16. azureDisk
17. vsphereVolume
18. Quobyte
19. PortworxVolume
20. ScaleIO
21. FlexVolume
22. StorageOS
23. local

注意：这些volume并非全部都是提供持久化的，比如emptyDir、secret、gitRepo等，这些volume会随着Pod的消亡而消失。

#### emptyDir

如果Pod设置了emptyDir类型的Volume，Pod被分配到Node上的时候，会创建emptyDir，只要Pod运行在Node上，emptyDir都会存在（容器挂掉不会导致emptyDir丢失数据），但是如果Pod从Node上被删除（Pod被删除，或者Pod发生迁移），emptyDir也会被删除，并且永久丢失。

```
apiVersion: v1
kind: Pod
metadata:
    name: test-pd
spec:
    containers:
    - image: gcr.io/google_containers/test-webserver
      name: test-container
      volumeMounts:
      - mountPath: /cache
        name: cache-volume
    volumes:
    - name: cache-volume
      emptyDir: {}
```

#### hostPath

hostPath允许挂载Node上的文件系统到Pod里面去。如果Pod需要使用Node上的文件，可以使用hostPath。

```
apiVersion: v1
kind: Pod
metadata:
    name: test-pd-hostPath
spec:
    containers:
    - image: gcr.io/google_containers/test-webserver
      name: test-container
      volumeMounts:
      - mountPath: /test-pd-hostPath
        name: test-volume
    volumes:
    - name: test-volume
      hostPath:
          path: /data
```