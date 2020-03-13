---
layout:     post
title:      Kubernetes(一)之基础介绍
subtitle:   Kubernetes 基础概念
date:       2020-03-13
author:     Frederick
header-img: img/dushu.jpg
catalog: true
tags:
    - Kubernetes
---


# Kubernetes基础介绍

# 简介
---
Kubernetes是一个开源的，用于管理云平台中多个主机上的容器化的应用，Kubernetes的目标是让部署容器化的应用简单并且高效（powerful）,Kubernetes提供了应用部署，规划，更新，维护的一种机制。

Kubernetes一个核心的特点就是能够自主的管理容器来保证云平台中的容器按照用户的期望状态运行着（比如用户想让apache一直运行，用户不需要关心怎么去做，Kubernetes会自动去监控，然后去重启，新建，总之，让apache一直提供服务），管理员可以加载一个微型服务，让规划器来找到合适的位置，同时，Kubernetes也系统提升工具以及人性化方面，让用户能够方便的部署自己的应用（就像canary deployments）。

在Kubenetes中，所有的容器均在Pod中运行,一个Pod可以承载一个或者多个相关的容器，在后边的案例中，同一个Pod中的容器会部署在同一个物理机器上并且能够共享资源。一个Pod也可以包含O个或者多个磁盘卷组（volumes）,这些卷组将会以目录的形式提供给一个容器，或者被所有Pod中的容器共享，对于用户创建的每个Pod,系统会自动选择那个健康并且有足够容量的机器，然后创建类似容器的容器,当容器创建失败的时候，容器会被node agent自动的重启,这个node agent叫kubelet,但是，如果是Pod失败或者机器，它不会自动的转移并且启动，除非用户定义了 replication controller。

### Kubernetes主要由以下几个核心组件组成
 - **Etcd**
保存了整个集群的状态,所有master的持续状态都存在etcd的一个实例中。这可以很好地存储配置数据。因为有watch(观察者)的支持，各部件协调中的改变可以很快被察觉。

 - **Api server**
 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制.它提供Kubernetes API （PS:官方 英文）的服务。这个服务试图通过把所有或者大部分的业务逻辑放到不两只的部件中从而使其具有CRUD特性。它主要处理REST操作，在etcd中验证更新这些对象（并最终存储）。

 - **Controller Manager**
 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等.

 - **Scheduler**
 负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上.

 - **Kubelet**
 负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理.

 - **Container Runtime**
 负责镜像管理以及Pod和容器的真正运行（CRI.）

 - **Kube-Proxy**
 负责为Service提供cluster内部的服务发现和负载均衡.

### Kubernetes 集群架构图
![Untitled Diagram (1).png](https://upload-images.jianshu.io/upload_images/17904159-eeb29f3f177b008d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###  Kubernetes的特点

- 1、可移植：支持公有云，私有云，混合云，多重云（multi-cloud）
- 2、可扩展：模块化, 插件化, 可挂载, 可组合
- 3、自动化：自动部署，自动重启，自动复制，自动伸缩/扩展

### Kubernetes的应用范围

- 多个进程（作为容器运行）协同工作。（Pod）
- 存储系统挂载
- Distributing secrets
- 应用健康检测
- 应用实例的复制
- Pod自动伸缩/扩展
- Naming and discovering
- 负载均衡
- 滚动更新
- 资源监控
- 日志访问
- 调试应用程序
- 提供认证和授权


**博客著作权归本作者所有，任何形式的转载都请联系作者获得授权并注明出处。**