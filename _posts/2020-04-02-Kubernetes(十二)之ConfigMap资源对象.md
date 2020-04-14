---
layout:     post
title:      Kubernetes(十二)之ConfigMap资源对象
subtitle:   
date:       2020-04-02
author:     Frederick
header-img: img/k8s.png
catalog: true
tags:
    - Kubernetes
---
# ConfigMap资源对象

## 介绍
ConfigMap对象实现了将配置文件从容器镜像中解耦，从而增强了容器应用的可移植性。简而言之，一个ConfigMap就是一系列配置数据的集合(如单个属性，或粗粒度信息如整个配置文件或JSON对象)，这些数据可“注入”到Pod对象中，并为容器应用所使用，注入方式有挂载为存储卷和传递为环境变量两种。

ConfigMap 对象将配置数据以键值对的形式存储，这些数据可以在Pod对象中使用或者为系统组件提供配置。

## ConfigMap使用方式

- 填充环境变量
- 设置容器内的命令行参数
- 填充卷的配置文件

## ConfigMap创建方式

- 直接在命令行中指定configmap参数创建，即--from-literal

- 指定文件创建，即将一个配置文件创建为一个ConfigMap--from-file=<文件>

- 指定目录创建，即将一个目录下的所有配置文件创建为一个ConfigMap，--from-file=<目录>

- 先写好标准的configmap的yaml文件，然后kubectl create -f 创建

## Yaml创建方式示例

### 场景介绍

在Kubernetes中部署Elasticsearch集群创建时，需要配置集群名称以及集群待加入节点信息，和设置集群master最少节点要求等配置，使用confmap来作为填充卷方式让Pod进行挂载容器内的文件路径/usr/share/elasticsearch/config/elasticsearch.yml，来提供elasticsearch.yml配置文件内容。

### ConfigMap Yaml

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: "es-config"
    data:
      elasticsearch.yml: |
        cluster.name: ${CLUSTER_NAME}
        network.host: "0.0.0.0"
        bootstrap.memory_lock: false
        node.master: ${NODE_MASTER}
        discovery.zen.ping.unicast.hosts: ${DISCOVERY_ZEN_PING_UNICAST_HOSTS}
        discovery.zen.minimum_master_nodes: ${DISCOVERY_ZEN_MINIMUM_MASTER_NODES}

###  Pod 局部定义YAML


    spec:
      initContainers:
        - name: fix-permissions
          image: "{{ .ImageBusyBox }}"
          command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
          securityContext:
            privileged: true
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
        - name: increase-vm-max-map
          image: "{{ .ImageBusyBox }}"
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
        - name: increase-fd-ulimit
          image: "{{ .ImageBusyBox }}"
          command: ["sh", "-c", "ulimit -n 65536"]
          securityContext:
            privileged: true
      containers:
        - name: elasticsearch
          image: "{{ .ImageEs}}"
          imagePullPolicy: Always
          env:
            - name: CLUSTER_NAME
              value: "sb-{{ .InstanceID }}-es-cluster"
            - name:  NODE_MASTER
              value: "true"
            - name: ES_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
            - name: DISCOVERY_ZEN_PING_UNICAST_HOSTS
              value: "sb-{{ .InstanceID }}-es-cluster-discovery"
            - name: DISCOVERY_ZEN_MINIMUM_MASTER_NODES
              value: "{{ .MasterMinNumber }}"
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
            - mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
              name: elasticsearch-config
              subPath: elasticsearch.yml
          resources:
            limits:
              memory: "{{ .ContainerMemory }}Gi"
              cpu: "{{ .ContainerCPU }}"
      nodeSelector:
        kubernetes.io/hostname: "{{ .TempHost }}"
      volumes:
        - name: data
          hostPath:
            path: "{{ .TempDataPath}}"
        - name: elasticsearch-config
          configMap:
            name: "es-config"

volumeMounts:将容器内要替换的配置文件路径/usr/share/elasticsearch/config/elasticsearch.yml设置为挂载点。将configmap设置成挂载卷的防止进行使用。

### 容器内部查看

    [root@sb-bsi-es-12161721-33-es-instanceid-0-0 config]# pwd
    /usr/share/elasticsearch/config
    [root@sb-bsi-es-12161721-33-es-instanceid-0-0 config]# ls
    elasticsearch.keystore  jvm.options        role_mapping.yml  users
    elasticsearch.yml       log4j2.properties  roles.yml         users_roles
    [root@sb-bsi-es-12161721-33-es-instanceid-0-0 config]# cat elasticsearch.yml
    cluster.name: ${CLUSTER_NAME}
    network.host: "0.0.0.0"
    bootstrap.memory_lock: false
    node.master: ${NODE_MASTER}
    discovery.zen.ping.unicast.hosts: ${DISCOVERY_ZEN_PING_UNICAST_HOSTS}
    discovery.zen.minimum_master_nodes: ${DISCOVERY_ZEN_MINIMUM_MASTER_NODES}
    [root@sb-bsi-es-12161721-33-es-instanceid-0-0 config]#

通过运行起来的服务我们进容器中查看目标挂载文件，已经从configmap设置的配置文件挂载进来了。
