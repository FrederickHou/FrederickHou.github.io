---
layout:     post
title:      Elasticsearch (三)容器化方案
subtitle:   Kubernetes 编排部署Elasticsearch以及Kibana
date:       2020-03-24
author:     Frederick
header-img: img/k8s.png
catalog: true
tags:
    - ElasticSearch
    - Kubernetes
---


# Kubernetes 编排部署Elasticsearch以及Kibana

## Elasticsearch介绍
请参考：[ElasticSearch(一)基础介绍](https://www.frederickhou.com/2020/03/17/ElasticSearch-%E4%B8%80-%E5%9F%BA%E7%A1%80%E4%BB%8B%E7%BB%8D/)

## Kibana介绍

Kibana是一个开源的分析与可视化平台，设计出来用于和Elasticsearch一起使用的。你可以用kibana搜索、查看存放在Elasticsearch中的数据。Kibana与Elasticsearch的交互方式是各种不同的图表、表格、地图等，直观的展示数据，从而达到高级的数据分析与可视化的目的。
Elasticsearch、Logstash和Kibana这三个技术就是我们常说的ELK技术栈，可以说这三个技术的组合是大数据领域中一个很巧妙的设计。一种很典型的MVC思想，模型持久层，视图层和控制层。Logstash担任控制层的角色，负责搜集和过滤数据。Elasticsearch担任数据持久层的角色，负责储存数据。而我们这章的主题Kibana担任视图层角色，拥有各种维度的查询和分析，并使用图形化的界面展示存放在Elasticsearch中的数据。

Kibana 的docker镜像运用 Centos:7作为基础镜像。发布过的相关Docker镜像包可以在www.docker.elastic.co官网进行查。笔者选用**6.8.7**版本进行安装演示。

## 创建Elasticsearch服务
### 创建Elasticsearch  Service
- **创建Elasticsearch集群节点发现服务（NodePort模式）**

    **问题**：es该如何发现其他节点？
    考虑到es的discovery.zen.ping.unicast.hosts参数可指定多IP域名，那么：

    - 为es创建headless service。
    - 指定该参数为service名。

    即可让es发现其他节点。

        apiVersion: v1
        kind: Service
        metadata:
        labels:
            app: es-cluster-discovery
        name: es-cluster-discovery
        spec:
        selector:
            app: es-node-instanceid
        ports:
            - port: 9300
            targetPort: 9300
    
- **创建Elasticsearch集群节点访问服务**

        apiVersion: v1
        kind: Service
        metadata:
        labels:
            app: es-cluster-service
        name: es-cluster-service
        spec:
        selector:
            app: es-node-instanceid
        type: NodePort
        ports:
            - port: 9200
            targetPort: 9200
创建结果如下：

    # kubectl get service -n local-pv -o wide
    NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE     SELECTOR
    es-cluster-discovery   ClusterIP   10.101.18.179    <none>        9300/TCP         22h     app=es-node-instanceid
    es-cluster-service     NodePort    10.106.56.69     <none>        9200:30709/TCP   7h5m    app=es-node-instanceid

### 创建Elasticsearch  Statefulset

#### 参数说明：
- **cluster.name**：集群名称，相同集群名称的节点会协调选举MASTER以及识别加入集群。
- **discovery.zen.ping.unicast.hosts**： 集群发现其他节点，或者其他节点通过此服务加入集群。
- **discovery.zen.minimum_master_nodes**： 构成集群的最少master节点数。一般设置为n/2+1个，防止“脑裂”现象。

#### 创建服务

    apiVersion: apps/v1beta1
    kind: StatefulSet
    metadata:
      name: es-instanceid
      labels:
        app: es-node-instanceid
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: es-node-instanceid
      #noted: serviceName is the real service name
      serviceName: "es-cluster-service"
      template:
        metadata:
          labels:
            app: es-node-instanceid
          annotations:
            scheduler.alpha.kubernetes.io/critical-pod: ''
        spec:
          securityContext:
            fsGroup: 1000
          initContainers:
          - name: fix-permissions
            image: docker.io/busybox:latest
            command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
            securityContext:
              privileged: true
            volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
          - name: increase-vm-max-map
            image: docker.io/busybox:latest
            command: ["sysctl", "-w", "vm.max_map_count=262144"]
            securityContext:
              privileged: true
          - name: increase-fd-ulimit
            image: docker.io/busybox:latest
            command: ["sh", "-c", "ulimit -n 65536"]
            securityContext:
              privileged: true
          containers:
          - name: elasticsearch
            image: docker.elastic.co/elasticsearch/elasticsearch:6.8.7
            imagePullPolicy: Always
            env:
            - name: "cluster.name"
              value: "elasticsearch-cluster"
            - name: "ES_JAVA_OPTS"
              value: "-Xms512m -Xmx512m"
            - name: "discovery.zen.ping.unicast.hosts"
              value: "es-cluster-discovery"
            - name: "discovery.zen.minimum_master_nodes"
              value: "2"
           # - name: "bootstrap.memory_lock"
           #   value: "true"
            ports:
            - containerPort: 9200
              protocol: TCP
              name: discovery
            - containerPort: 9300
              protocol: TCP
              name: transport
            volumeMounts:
                - mountPath: /usr/share/elasticsearch/data
                  name: data
                - mountPath: /usr/share/elasticsearch/logs
                  name: data
            #resources:
             # limits:
               # cpu: 25m
               # memory: 3Gi
          nodeSelector:
            kubernetes.io/hostname: cent165
          volumes:
             - name: data
               #emptyDir: {}
               hostPath:
                  path: "/ocdf_local_pv_vg0/es"

**注**：笔者为了演示创建了两个节点，正式使用必须遵循规则防止“脑裂”现象。
创建结果如下：

    # kubectl get pod -n local-pv -o wide
    NAME                      READY   STATUS    RESTARTS   AGE   IP                NODE      NOMINATED NODE   READINESS GATES
    es-instanceid-1-0         1/1     Running   0          22h   100.81.251.208    cent166   <none>           <none>
    es-instanceid-2-0         1/1     Running   0          22h   100.110.104.217   cent165   <none>           <none>

查看集群健康状态：green表示健康

    #curl http://10.1.234.164:30709/_cluster/health?pretty
    {
    "cluster_name" : "elasticsearch-cluster",
    "status" : "green",
    "timed_out" : false,
    "number_of_nodes" : 2,
    "number_of_data_nodes" : 2,
    "active_primary_shards" : 5,
    "active_shards" : 10,
    "relocating_shards" : 0,
    "initializing_shards" : 0,
    "unassigned_shards" : 0,
    "delayed_unassigned_shards" : 0,
    "number_of_pending_tasks" : 0,
    "number_of_in_flight_fetch" : 0,
    "task_max_waiting_in_queue_millis" : 0,
    "active_shards_percent_as_number" : 100.0
    }

查看集群nodes:

    #curl http://10.1.234.164:30709/_cat/nodes?v
    ip              heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
    100.110.104.217           60          94   2    0.33    0.13     0.07 mdi       *      1mSnPgt
    100.81.251.208            62          97   3    0.27    0.19     0.20 mdi       -      clNDNoX

查看集群masters:

    #curl http://10.1.234.164:30709/_cat/master?v
    id                     host            ip              node
    1mSnPgtDS8O3_FsmX-PGLA 100.110.104.217 100.110.104.217 1mSnPgt

## 创建Kibana服务
### 创建Kibana  Service（NodePort模式）

    apiVersion: v1
    kind: Service
    metadata:
    name: kibana
    labels:
        app: kibana
    spec:
    type: NodePort
    ports:
    - port: 5601
    selector:
        app: kibana

创建结果如下：

    # kubectl get service -n local-pv -o wide
    kibana                 NodePort    10.107.131.227   <none>        5601:31920/TCP   6h49m   app=kibana

### 创建Kibana 服务
#### 参数说明：
- **ELASTICSEARCH_URL**: 连接es的url，如："http://es-cluster-service:9200" 其中“es-cluster-service”是es集群的9200 service name。
- **elasticsearch.hosts**：es的host列表，一般也是设置成es 9200 service name。通过服务名字可以解析出对应节点的IP。

#### 创建服务：

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kibana
      labels:
        app: kibana
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: kibana
      template:
        metadata:
          labels:
            app: kibana
        spec:
          containers:
          - name: kibana
            image: docker.elastic.co/kibana/kibana:6.8.7
            resources:
              limits:
                cpu: 1000m
              requests:
                cpu: 100m
            env:
              - name: "ELASTICSEARCH_URL"
                value: "http://es-cluster-service:9200"
              - name: "elasticsearch.hosts"
                value: "es-cluster-service"
              - name: "elasticsearch.username"
                value: "admin"
              - name: "elasticsearch.password"
                value: "admin"
            ports:
            - containerPort: 5601
    
创建结果如下：

    # kubectl get pod -n local-pv -o wide
    kibana-76969bdf57-xzxc9   1/1     Running   0          73m   100.81.251.214    cent166   <none>           <none>

#### 浏览器访问

浏览器地址中输入：http://10.1.234.164:31920 如下图所示：

![](https://github.com/FrederickHou/FrederickHou.github.io/blob/master/img/kibana1.jpg?raw=true)

![](https://github.com/FrederickHou/FrederickHou.github.io/blob/master/img/kibana2.jpg?raw=true)

es集群状态：
![](https://github.com/FrederickHou/FrederickHou.github.io/blob/master/img/kibana3.jpg?raw=true)