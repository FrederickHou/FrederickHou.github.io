---
layout:     post
title:      Kubernetes(十三)之Service资源对象
subtitle:   
date:       2020-04-14
author:     Frederick
header-img: img/k8s.png
catalog: true
tags:
    - Kubernetes
---
# Service资源对象

## 介绍

在生产环境中，我们不能期望Pod一直是健壮的。假设Pod的容器很可能因为各种原因发送故障而挂掉。Deployment等Controller会通过动态创建和销毁Pod来保证应用整体的健壮性。

在创建的每个Pod中，都有自己的IP地址，当Controller用新Pod替代发生故障的Pod时，新Pod会分配到新的IP地址。那么这样就产生一个问题，如果在Pod是对外提供服务的，如HTTP服务，它们在重新创建时，IP地址也就发生了变化，那么客户如何找到并访问这个服务呢？此时Kubernetes就会有了Service。

Service是一个抽象概念，定义了一个服务的多个pod逻辑合集和访问pod的策略，一般把Service称为微服务。借助 Service，应用可以方便的实现服务发现与负载均衡，并实现应用的零宕机升级。Service 通过标签来选取服务后端，一般配合 Replication Controller 或者 Deployment 来保证后端容器的正常运行。这些匹配标签的 Pod IP 和端口列表组成 endpoints，由 kube-proxy 负责将服务 IP 负载均衡到这些 endpoints 上。

## Service类型---发布服务

- **ClusterIP：** 默认类型，自动分配一个仅 cluster 内部可以访问的虚拟 IP。
- **NodePort：** 在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口，这样就可以通过 NodeIP:NodePort 来访问该服务。如果 kube-proxy 设置了 – - nodeport-addresses=10.240.0.0/16（v1.10 支持），那么仅该 NodePort 仅对设置在范围内的 IP 有效。
- **LoadBalancer：** 在 NodePort 的基础上，借助 cloud provider 创建一个外部的负载均衡器，并将请求转发到 :NodePort
- **ExternalName：** 将服务通过 DNS CNAME 记录方式转发到指定的域名（通过 spec.externlName 设定）。需要 kube-dns 版本在 1.7 以上。


## 网络代理模式

- **userspace** ：client先请求serviceip，经由iptables转发到kube-proxy上之后再转发到pod上去。这种方式效率比较低。

  ![userspace.jpg](https://d33wubrfki0l68.cloudfront.net/e351b830334b8622a700a8da6568cb081c464a9b/13020/images/docs/services-userspace-overview.svg)

- **iptables** ：lient请求serviceip后会直接转发到pod上。这种模式性能会高很多。kube-proxy就会负责将pod地址生成在node节点iptables规则中。

  ![iptables.jpg](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg)

- **ipvs** ：它是直接有内核中的ipvs规则来接受Client Pod请求，并处理该请求,再有内核封包后，直接发给指定的Server Pod。

  ![ipvs.jpg](https://d33wubrfki0l68.cloudfront.net/2d3d2b521cf7f9ff83238218dac1c019c270b1ed/9ac5c/images/docs/services-ipvs-overview.svg)

**注：**

　以上不论哪种，kube-proxy都通过watch的方式监控着kube-APIServer写入etcd中关于Pod的最新状态信息,它一旦检查到一个Pod资源被删除了或新建，它立即将这些变化，反应在iptables 或 ipvs规则中，以便iptables和ipvs在调度Clinet Pod请求到Server Pod时，不会出现Server Pod不存在的情况。

## 创建服务

    apiVersion: v1
    kind: Service
    metadata:
      name: "maxscale"
      labels:
        mariadb: "mariadb"
        entrypoint.mariadb: "mariadb"
    spec:
      type: NodePort
      ports:
        - name: maxscale-readwrite
          port: 4006
          targetPort: 4006
          nodePort: 31783
        - name: maxscale-readonly
          port: 4008
          targetPort: 4008
      selector:
        maxscale.mariadb: "mariadb"

### yaml说明：

- **selector：** 指定了为哪一个标签的app进行负载均衡。
- **nodePort：** 节点数监听的端口。
- **targetPort:** Pod监听的端口。

## 服务发现

### headless service(无头service)

**headless service:** 没有ClusterIP的service, 它仅有一个service name.这个服务名解析得到的不是service的集群IP，而是Pod的IP,当其它人访问该service时，将直接获得Pod的IP,进行直接访问。

### 示例

    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: "es-cluster-discovery"
      name: "es-cluster-discovery"
    spec:
      selector:
        app: "es-node-instanceid"
      ports:
        - port: 9300
          name: discovery
          targetPort: 9300

以上es-cluster-discovery服务是用作es集群通过9300来发现其他待加入节点的Pod IP。es的“DISCOVERY_ZEN_PING_UNICAST_HOSTS”参数是待加入集群节点列表，此时可以将"es-cluster-discovery"作为该参数的值，进行解析，直接获取到Pod的Ip。具体详见 [Elasticsearch容器化](https://www.frederickhou.com/2020/03/24/Elasticsearch-(%E4%B8%89)%E5%AE%B9%E5%99%A8%E5%8C%96%E6%96%B9%E6%A1%88/)。




参考：https://kubernetes.io/zh/docs/concepts/services-networking/service/