---
layout:     post
title:      Kubernetes(八)亲和性与反亲和性
subtitle:   
date:       2020-03-27
author:     Frederick
header-img: img/k8s.png
catalog: true
tags:
    - Kubernetes
---

# 亲和性与反亲和性

nodeSelector 提供了一个非常简单的方式，将 Pod 调度限定到包含特定标签的节点上。亲和性与反亲和性（affinity / anti-affinity）特性则极大地扩展了限定的表达方式。主要的增强点在于：

表达方式更加有效（不仅仅是多个精确匹配表达式的“和”关系）可以标识该规则为“soft” / “preference” （软性的、偏好的）而不是 hard requirement（必须的），此时，如果调度器发现该规则不能被满足，Pod 仍然可以被调度
可以对比节点上（或其他拓扑域 topological domain）已运行的其他 Pod 的标签，而不仅仅是节点自己的标签，此时，可以定义类似这样的规则：某两类 Pod 不能在同一个节点（或拓扑域）上共存

例如控制器DaemonSet 是决定了POD在集群中每个可调度节点上都保证运行1个副本存。但是对于DeployMent、ReplicatSet、StatefulSet。这种是保证在集群中可调度节点上运行一定数量Rplicas的Pod。对于只想在满足限制条件的节点上且运行一个POD那么久需要用到Pod的反亲和来实现。


## 节点亲和性

节点亲和性（node affinity）的概念与 nodeSelector 相似，可以基于节点的标签来限定 Pod 可以被调度到哪些节点上。

当前支持两种类型的节点亲和性：

- **requiredDuringSchedulingIgnoredDuringExecution** （hard，目标节点必须满足此条件 

- **preferredDuringSchedulingIgnoredDuringExecution** （soft，目标节点最好能满足此条件）。

名字中 **IgnoredDuringExecution** 意味着：如果 Pod 已经调度到节点上以后，节点的标签发生改变，使得节点已经不再匹配该亲和性规则了，Pod 仍将继续在节点上执行（这一点与 **nodeSelector** 相似）。将来，Kubernetes 将会提供 requiredDuringSchedulingRequiredDuringExecution 这个选项，该选项与 requiredDuringSchedulingIgnoredDuringExecution 相似，不同的是，当节点的标签不在匹配亲和性规则之后，Pod 将被从节点上驱逐。

requiredDuringSchedulingIgnoredDuringExecution 的一个例子是，只在 Intel CPU 上运行该 Pod，preferredDuringSchedulingIgnoredDuringExecution 的一个例子是，尽量在高可用区 XYZ 中运行这个 Pod，但是如果做不到，也可以在其他地方运行该 Pod。

**注：** 如果某个 Pod 同时指定了 nodeSelector 和 nodeAffinity，则目标节点必须同时满足两个条件，才能将 Pod 调度到该节点上。

如果为 nodeAffinity 指定多个 nodeSelectorTerms，则目标节点只需要满足任意一个 nodeSelectorTerms 的要求，就可以将 Pod 调度到该节点上。

如果为 nodeSelectorTerms 指定多个 matchExpressions，则目标节点必须满足所有的 matchExpressions 的要求，才能将 Pod 调度到该节点上。

当 Pod 被调度到某节点上之后，如果移除或者修改节点的标签，Pod 将仍然继续在节点上运行。换句话说，节点亲和性规则只在调度该 Pod 时发生作用。

**示例说明：**

    apiVersion: v1
    kind: Pod
    metadata:
      name: node-affinity
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/appmanager
                operator: In
                values:
                - true
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: another-node-label-key
                operator: In
                values:
                - another-node-label-value
      containers:
      - name: node-affinity
        image: docker.io/busybox:latest

此处的亲和性规则表明，该 Pod 只能被调度到包含 key 为 kubernetes.io/appmanager 且 value 为 true  的标签的节点上。此外，如果节点已经满足了前述条件，将优先选择包含 key 为 another-node-label-key 且 value 为 another-node-label-value 的标签的节点。

## Pod亲和性与反亲和性

Pod之间的亲和性与反亲和性（inter-pod affinity and anti-affinity）可以基于已经运行在节点上的 Pod 的标签（而不是节点的标签）来限定 Pod 可以被调度到哪个节点上。此类规则的表现形式是

**示例说明：**

    apiVersion: extensions/v1beta1
    kind: ReplicaSet
    metadata:
      name: pod-antiaffinity
    spec:
      replicas: 5
      selector:
        matchLabels:
          app: pod-antiaffinity
      template:
        metadata:
          labels:
            app: pod-antiaffinity
        spec:
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchExpressions:
                      - key: app
                        operator: In
                        values:
                          - pod-antiaffinity
                  topologyKey: "kubernetes.io/hostname"
          containers:
            - name: pod-antiaffinity
              image: docker.io/busybox:latest
              command: ["sh", "-c", "sleep  3600"]
              resources:
                limits:
                  memory: 1Gi
                  cpu: 1
          nodeSelector:
            kubernetes.io/appmanager: "true"

笔者此处举一个特殊例子的场景。Pod控制器ReplicaSet是保证在集群中启动一定数量的Pod。但是我们又不想让其在同一个节点上启动多个副本，此时我们用Pod反亲和性就可以得到解决，如上述例子。
- Pod 反亲和性规则要求，该 Pod 最好不要被调度到已经运行了包含 key 为 app 且 value 为 pod-antiaffinity 的标签的 Pod 的节点上，或者更准确地说，必须满足如下条件：
    - 如果 topologyKey 是 kubernetes.io/hostname，则，Pod不能被调度到同一个 zone 中的已经运行了包含标签 app: pod-antiaffinity 的节点上。
