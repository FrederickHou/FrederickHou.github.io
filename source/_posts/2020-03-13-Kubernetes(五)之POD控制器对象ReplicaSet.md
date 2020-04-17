---
layout:     post
title:      Kubernetes(五)之POD控制器对象ReplicaSet
subtitle:   Kubernetes 控制器对象
date:       2020-03-30
author:     Frederick
header-img: img/k8s.png
categories:
    - 技术笔记
    - Kubernetes 笔记
tags:
    - Kubernetes
toc: true
cover: https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=2389716159,3815266060&fm=26&gp=0.jpg
---

# POD控制器对象ReplicaSet
## ReplicaSet介绍

### 什么是ReplicaSet？

ReplicaSet是下一代复本控制器，是Replication Controller（RC）的升级版本。ReplicaSet和 Replication Controller之间的唯一区别是对选择器的支持。ReplicaSet支持labels user guide中描述的set-based选择器要求， 而Replication Controller仅支持equality-based的选择器要求。

### 如何使用ReplicaSet？

大多数kubectl 支持Replication Controller 命令的也支持ReplicaSets。rolling-update命令除外，如果要使用rolling-update，请使用Deployments来实现。

　　虽然ReplicaSets可以独立使用，但它主要被 **Deployments**用作pod 机制的创建、删除和更新。当使用Deployment时，你不必担心创建pod的ReplicaSets，因为可以通过Deployment实现管理ReplicaSets

### 何时使用ReplicaSet？

**ReplicaSet能确保运行指定数量的pod**。然而，Deployment 是一个更高层次的概念，它能管理ReplicaSets，并提供对pod的更新等功能。因此，我们建议你使用Deployment来管理ReplicaSets，除非你需要自定义更新编排。

　　这意味着你可能永远不需要操作ReplicaSet对象，而是使用Deployment替代管理 。

## ReplicaSet对象定义

- **apiVersion**: app/v1  版本
- **kind**:  ReplicaSet  类型
- **metadata**:  元数据
- **spec**:   期望状态
    - **minReadySeconds**: 应为新创建的pod准备好的最小秒数
    - **replicas**: 副本数； 默认为1
    - **selector**: 标签选择器
    - **template**: pod模板
        - **metadata**: pod模板中的元数据
        - **spec**: pod模板中的期望状态

## ReplicaSet创建示例

### Yaml示例

    apiVersion: apps/v1
    kind: ReplicaSet
    metadata:
      name: myapp
      namespace: default
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: myapp
      template:
        metadata:
          name: myapp-pod
          labels:
            app: myapp
        spec:
          containers:
          - name: myapp-container
            image: docker.io/busybox:latest
            command: ["sh","-c","sleep 3600"]

### 创建结果

- 创建

        # kubectl create -f rs-demo.yaml
        replicaset.apps/myapp created
- 查看Replicaset状态

        # kubectl get rs
        NAME      DESIRED   CURRENT   READY     AGE
        myapp     2         2         2         23s

- 查看pod 状态

        # kubectl get pods
        NAME          READY     STATUS    RESTARTS   AGE
        myapp-r4ss4   1/1       Running   0          25s
        myapp-zjc5l   1/1       Running   0          26s

    生产的pod原则(多退少补)：根据replicas配置保证生成相应数量的pod。

## ReplicaSet扩所容

- 使用edit 修改replicaset 配置，将副本数改为5；即可实现动态扩容

        # kubectl edit rs myapp
        ... ...
        spec:
        replicas: 5
        ... ...
        replicaset.extensions/myapp edited
- 查看结果 

        # kubectl get pods
        NAME          READY     STATUS    RESTARTS   AGE
        myapp-bck7l   1/1       Running   0          16s
        myapp-h8cqr   1/1       Running   0          16s
        myapp-hfb72   1/1       Running   0          6m
        myapp-r4ss4   1/1       Running   0          9m
        myapp-vvpgf   1/1       Running   0          16s

## ReplicaSet在线升级

原理同扩所容同样，进行编辑replicaset 对containers对应的image进行版本升级修改。**注**：修改完并没有升级需删除pod，再自动生成新的pod时，就会升级成功；即可以实现灰度发布：删除一个，会自动启动一个版本升级成功的pod
