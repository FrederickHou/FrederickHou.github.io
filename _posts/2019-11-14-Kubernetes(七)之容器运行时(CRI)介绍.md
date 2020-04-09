---
layout:     post
title:      Kubernetes(七)之容器运行时(CRI)介绍
subtitle:   
date:       2019-11-14
author:     Frederick
header-img: img/k8s.png
catalog: true
tags:
    - Kubernetes
---


### 容器运行时简介(CRI)

容器运行时接口（Container Runtime Interface，简称 CRI）是 Kubernetes v1.5 引入的容器运行时接口，它将 Kubelet 与容器运行时解耦，将原来完全面向 Pod 级别的内部接口拆分成面向 Sandbox 和 Container 的 gRPC 接口，并将镜像管理和容器管理分离到不同的服务。且到笔者目前的时间段CRI是Alpha版本。

### 容器运行时总流程(CRI)

Kubelet与容器运行时通信（或者是CRI插件填充了容器运行时）时，Kubelet就像是客户端，而CRI插件就像对应的服务器。它们之间可以通过Unix 套接字或者gRPC框架进行通信。

![image.png](https://upload-images.jianshu.io/upload_images/17904159-401ca5f0fa105800.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

protocol buffers API包含了两个gRPC服务：ImageService和RuntimeService。ImageService提供了从镜像仓库拉取、查看、和移除镜像的RPC。RuntimeSerivce包含了Pods和容器生命周期管理的RPC，以及跟容器交互的调用(exec/attach/port-forward)。一个单块的容器运行时能够管理镜像和容器（例如：Docker和Rkt），并且通过同一个套接字同时提供这两种服务。这个套接字可以在Kubelet里通过标识–container-runtime-endpoint和–image-service-endpoint进行设置。
采用CRI后kubelet 架构图如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/17904159-3e069487e4c290a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 参考
[https://yq.aliyun.com/articles/679819](https://yq.aliyun.com/articles/679819)
