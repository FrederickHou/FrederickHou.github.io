---
layout:     post
title:      Kubernetes(三)之资源对象POD和JOB
subtitle:   Kubernetes POD和JOB
date:       2020-03-13
author:     Frederick
header-img: img/dushu.jpg
catalog: true
tags:
    - Kubernetes
---
# Kubernetes POD和JOB
## pod
Pod是K8S的最小操作单元，一个Pod可以由一个或多个容器组成；
整个K8S系统都是围绕着Pod展开的，比如如何部署运行Pod、如何保证Pod的数量、如何访问Pod等。

### 特点
Pod是能够被创建、调度和管理的最小单元；每个Pod都有一个独立的IP；
一个Pod由一个或多个容器构成，并共享命名空间和共享存储等；Pod所有容器在同一个Node上；

#### 容器生命周期管理

对资源使用进行限制，resources(requests、limits)

对容器进行探测：livenessProbe

集群内的Pod之间都可以任意访问，这一般是通过一个二层网络来实现的。

### Pod与容器
在Docker中，容器是最小的处理单元，增删改查的对象是容器，容器是一种虚拟化技术，容器之间是隔离的，隔离是基于Linux Namespace实现的。

而在K8S中，Pod包含一个或者多个相关的容器，Pod可以认为是容器的一种延伸扩展，一个Pod也是一个隔离体，而Pod内部包含的一组容器又是共享的（包括PID、Network、IPC、UTS）。除此之外，Pod中的容器可以访问共同的数据卷来实现文件系统的共享。

### 资源请求与限制
创建Pod时，可以指定计算资源（目前支持的资源类型有CPU和内存），即指定每个容器的资源请求（Request）和资源限制（Limit），资源请求是容器所需的最小资源需求，资源限制则是容器不能超过的资源上限。关系是： 0<=request<=limit<=infinity


Pod的资源请求就是Pod中所有容器资源请求之和。K8S在调度Pod时，会根据Node中的资源总量（通过cAdvisor接口获得），以及该Node上已使用的计算资源，来判断该Node是否满足需求。

资源请求能够保证Pod有足够的资源来运行，而资源限制则是防止某个Pod无限制地使用资源，导致其他Pod崩溃。特别是在公有云场景，往往会有恶意软件通过抢占内存来攻击平台。

具体配置资源限制如下所示:

    resources:
    limits:
        memory: {{ .ContainerMemory }}Gi
        cpu: {{ .ContainerCPU }}


### 一pod多容器
Pod主要是在容器化环境中建立一个面向应用的“逻辑主机”模型，它可以包含一个或多个相互间紧密联系的容器。当其中任一容器异常时，该Pod也随之异常。

一pod多容器，让多个同应用的单一容器整合到一个类虚拟机中，使其所有容器共用一个vm的资源，提高耦合度，从而方便副本的复制，提高整体的可用性。

#### 一pod多容器的优势：

同个Pod下的容器之间能更方便的共享数据和通信，使用相同的网络命名空间、IP地址和端口区间，相互之间能通过localhost来发现和通信。

在同个Pod内运行的容器共享存储空间（如果设置），存储卷内的数据不会在容器重启后丢失，同时能被同Pod下别的容器读取。

相比原生的容器接口，Pod通过提供更高层次的抽象，简化了应用的部署和管理，不同容器提供不同服务。Pod就像一个管理横向部署的单元，主机托管、资源共享、协调复制和依赖管理都可以自动处理。


## Job
在有些场景下，是想要运行一些容器执行某种特定的任务，任务一旦执行完成，容器也就没有存在的必要了。在这种场景下，创建pod就显得不那么合适。于是就是了Job，Job指的就是那些一次性任务。通过Job运行一个容器，当其任务执行完以后，就自动退出，集群也不再重新将其唤醒。

从程序的运行形态上来区分，可以将Pod分为两类：长时运行服务（jboss、mysql等）和一次性任务（数据计算、测试）。RC创建的Pod都是长时运行的服务，Job多用于执行一次性任务、批处理工作等，执行完成后便会停止（status.phase变为Succeeded）。

### yaml配置参数说明
#### 重启策略
支持两种重启策略：

- **OnFailure**:在出现故障时其内部重启容器，而不是创建。

- **Never**:会在出现故障时创建新的，且故障job不会消失。

##### 设置超时
job执行超时时间可以通过spec.activeDeadlineSeconds来设置，超过指定时间未完成的job会以DeadlineExceeded原因停止

#### 并行
.spec.completions：这个job运行pod的总次数

.spec.parallelism：并发数，每次同时运行多少个pod

当completions少于parallelism，parallelism的值为completions

可以使用kubectl scale命令来增加或者减少completions的值

#### pod selector
job同样可以指定selector来关联pod。需要注意的是job目前可以使用两个API组来操作，batch/v1和extensions/v1beta1。当用户需要自定义selector时，使用两种API组时定义的参数有所差异。

使用batch/v1时，用户需要将jod的spec.manualSelector设置为true，才可以定制selector。默认为false。

使用extensions/v1beta1时，用户不需要额外的操作。因为extensions/v1beta1的spec.autoSelector默认为false，该项与batch/v1的spec.manualSelector含义正好相反。换句话说，使用extensions/v1beta1时，用户不想定制selector时，需要手动将spec.autoSelector设置为true。

例子


    apiVersion: batch/v1
    kind: Job
    metadata:
    labels:
    app: job
    project: lykops
    version: v1
    name: lykops-job
    namespace: default
    spec:
    completions: 50
    parallelism: 5
    template:
    metadata:
        labels:
        app: job
        job-name: lykops-job
        project: lykops
        version: v1
        name: lykops-job
    spec:
        containers:
        - command: ['sleep','60']
        image: web:apache
        name: lykops-job
        restartPolicy: Never

## POD和JOB的区别
job执行完后，不会自动启动一个新的pod，pod也不会被自动删除。
使用kubectl get pod无法显示执行完的job的pod，详细查看已运行过的job的pod需要添加参数—all-show或者-a，kubectl get pods -a。
