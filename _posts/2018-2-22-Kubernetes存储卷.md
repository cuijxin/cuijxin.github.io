---
layout:     post
title:      Kubernetes存储卷
subtitle:   《Kubernetes指南》书摘
date:       2018-02-22
author:     cjx
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - K8S
---

### Kubernetes存储卷

我们知道默认情况下容器的数据都是非持久化的，在容器消亡以后数据也跟着丢失，所以Docker提供了Volume机制以便将数据持久化存储。类似的Kubernetes提供了更强大的Volume机制和丰富的插件，解决了容器数据持久化和容器间共享数据的问题。

与Docker不同，Kubernetes Volume的声明周期与Pod绑定。
1. 容器挂掉后，Kubelet再次重启容器时，Volume的数据依然还在。
2. 而Pod删除时，Volume才会清理。数据是否丢失取决于具体的Volume类型，比如emptyDir的数据会丢失，而PV的数据则不会丢失。

### Volume类型

目前，Kubernetes支持以下Volume类型：
```
1. emptyDir                   9. rbd                         17. vsphereVolume               
2. hostPath                  10. cephfs                      18. Quobyte
3. gcePersistentDisk         11. gitRepo                     19. PortworxVolume
4. awsElasticBlockStore      12. secret                      20. ScaleIO
5. nfs                       13. persistentVolumeClaim       22. FlexVolume
6. iscsi                     14. downwardAPI                 23. StorageOS
7. flocker                   15. azureFileVolume             24. local
8. glusterfs                 16. azureDisk
```

注意，这些Volume并非全部都是持久化的，比如emptyDir、secret、gitRepo等，这些volume会随着Pod的消亡而消失。

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
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /data
```

#### NFS

NFS是Network File System的缩写，即网络文件系统。Kubernetes中通过简单地配置就可以挂载NFS到Pod中，而NFS中的数据是可以永久保存的，同时NFS支持同时写操作。
```
volumes:
- name: nfs
  nfs:
    # FIXME: use the right hostname
    server: 10.254.234.223
    path: "/"
```

#### gcePersistentDisk

gcePersistentDisk可以挂载GCE上的永久磁盘到容器，需要Kubernetes运行在GCE的VM中。
```
volumes:
- name: test-volume
  # This GCE PD must already exist.
  gcePersistentDisk:
    pdName: my-data-disk
    fsType: ext4
```

#### awsElasticBlockStore

awsElasticBlockStore可以挂载AWS上的EBS盘到容器，需要Kubernetes运行在AWS的EC2上。
```
volumes:
- name: test-volume
  # This AWS EBS volume must already exist.
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

#### gitRepo

gitRepo volume将git代码下拉到指定的容器路径中。
```
volumes:
- name: git-volume
  gitRepo:
    repository: "git@somewhere:me/my-git-repository.git"
    revision: "22f1d8406d464b0c0874075539c1f2e96c253775"
```

### 使用subPath

Pod的多个容器使用同一个Volume时，subPath非常有用
```
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
  containers:
  - name: mysql
    image: mysql
    volumeMounts:
    - mountPath: /var/lib/mysql
      name: site-data
      subPath: mysql
  - name: php
    image: php
    volumeMounts:
    - mountPath: /var/www/html
      name: site-data
      subPath: html
  volumes:
  - name: site-data
    persistentVolumeClaim:
      claimName: my-lamp-site-data 
```

### FlexVolume

如果内置的这些Volume不满足要求，则可以使用FlexVolume实现自己的Volume插件。注意要把volume plugin放到```/usr/libexec/kubernetes/kubelet-plugins/volume/exec/<vendor~driver>/<driver>```，plugin要实现```init/attach/detach/mount/umount```等命令（可参考lvm的示例）。
```
- name: test
  flexVolume:
    driver: "kubernetes.io/lvm"
    fsType: "ext4"
    options:
      volumeID: "vol1"
      size: "1000m"
      volumegroup: "kube_vg"
```

### Projected Volume

Projected volume将多个Volume源映射到同一个目录中，支持secret、downwardAPI和configMap。

```
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
          - key: username
            path: my-group/my-username
      - downardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "cpu_limit"
              resourceFieldRef:
                containerName: container-test
                resource: limits.cpu
      - configMap:
          name: myconfigmap
          items:
            - key: config
              path: my-group/my-config
```

### 本地存储限额

v1.7+支持对基于本地存储（如hostPath，emptyDir，gitRepo等）的容量进行调度限额，可以通过```feature-gates=LocalStorageCapacityIsolation=true```来开启这个特性。

为了支持这个特性，Kubernetes将本地存储分为两类：

1. ```storage.kubernetes.io/overlay```，即```/var/lib/docker```的大小。

2. ```storage.kubernetes.io/scratch```，即```/var/lib/kubelet```的大小。

Kubernetes根据```storage.kubernetes.io/scratch```的大小来调度本地存储空间，而根据```storage.kubernetes.io/overlay```来调度容器的存储。比如：
为容器请求64MB的可写层存储空间
