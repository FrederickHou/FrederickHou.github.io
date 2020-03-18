---
layout:     post
title:      ElasticSearch（二）安装部署
subtitle:   大数据 组件
date:       2020-03-18
author:     Frederick
header-img: https://upload-images.jianshu.io/upload_images/17904159-1e67e74d129675d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
catalog: true
tags:
    - ElasticSearch
---

# Elasticsearch  docker 安装部署

**Elasticsearch(ES)** 的docker镜像运用 [Centos:7](https://hub.docker.com/_/centos/)作为基础镜像。发布过的相关Docker镜像包可以在[www.docker.elastic.co](https://www.docker.elastic.co)官网进行查。源码托管在[Github](https://github.com/elastic/elasticsearch/blob/7.6/distribution/docker)上。

## 单节点安装

### 拉取镜像包
    docker pull docker.elastic.co/elasticsearch/elasticsearch:7.6.1

其他的版本镜像包可以去[www.docker.elastic.co](https://www.docker.elastic.co)进行下载。

### 运行一个单节点集群的服务

运行一个单节点集群服务用来开发或者测试。并且要指定单节点发现来绕过引导检查。详细引导检查请查看[引导检查](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html)

    docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.6.1


## 多节点集群安装

### 使用Docker Compose 启动一个多节点集群 

- **1 创建一个docker compose文件docker-compose.yml**

        version: '2.2'
        services:
          es01:
            image: docker.elastic.co/elasticsearch/elasticsearch:7.6.1
            container_name: es01
            environment:
              - node.name=es01
              - cluster.name=es-docker-cluster
              - discovery.seed_hosts=es02,es03
              - cluster.initial_master_nodes=es01,es02,es03
              - bootstrap.memory_lock=true
              - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
            ulimits:
              memlock:
                soft: -1
                hard: -1
            volumes:
              - data01:/usr/share/elasticsearch/data
            ports:
              - 9200:9200
            networks:
              - elastic
          es02:
            image: docker.elastic.co/elasticsearch/elasticsearch:7.6.1
            container_name: es02
            environment:
              - node.name=es02
              - cluster.name=es-docker-cluster
              - discovery.seed_hosts=es01,es03
              - cluster.initial_master_nodes=es01,es02,es03
              - bootstrap.memory_lock=true
              - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
            ulimits:
              memlock:
                soft: -1
                hard: -1
            volumes:
              - data02:/usr/share/elasticsearch/data
            networks:
              - elastic
          es03:
            image: docker.elastic.co/elasticsearch/elasticsearch:7.6.1
            container_name: es03
            environment:
              - node.name=es03
              - cluster.name=es-docker-cluster
              - discovery.seed_hosts=es01,es02
              - cluster.initial_master_nodes=es01,es02,es03
              - bootstrap.memory_lock=true
              - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
            ulimits:
              memlock:
                soft: -1
                hard: -1
            volumes:
              - data03:/usr/share/elasticsearch/data
            networks:
              - elastic

        volumes:
          data01:
            driver: local
          data02:
            driver: local
          data03:
            driver: local

        networks:
          elastic:
            driver: bridge

此Docker Compose 例子文件将创建一个3个节点的Elasticsearch集群。节点es01 监听 localhost:9200 并且在Docker网络中节点es02 和es03要和es01通信。

请记录此配置文件导出的端口号：9200，并且要在所有网络接口上配置Linux的iptables端口开放信息，这样Elasticsearch服务就有权限公开访问了。如果您不想公开端口9200，而是使用反向代理，请在docker-compose.yml文件中将9200：9200替换为127.0.0.1:9200:9200。 然后只能从主机本身访问Elasticsearch。

    #开发9200端口
    iptables -A INPUT -p tcp --dport 22 -j ACCEPT
    service iptables save

卷data01,data02,和data01直接存储着节点数据，没有用持久化存储，此种配置在重启服务后数据可能丢失

- **2 确保为Docker Engine分配了至少4GiB的内存**

    如果docker compose 没有安装请查看docs.docker.com网站进行[安装](https://docs.docker.com/compose/install)

        sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
        sudo chmod +x /usr/local/bin/docker-compose 
    查看版本

        # docker-compose --version
        docker-compose version 1.25.4, build 8d51620a


- **3 运行docker compose 来创建拉起集群**

        docker-compose up -d
    
    **报错：** 
    iptables: No chain/target/match by that name
    **原因：**
    docker服务启动时定义的自定义链
    DOCKER由于某种原因被清掉重启docker服务及可重新生成自定义链DOCKER
    **解决方案：**

        iptables -t filter -F

        iptables -t filter -X

        systemctl restart docker

    **docker 运行查看**

        # docker ps|grep es0
        86b91ba4732f        docker.elastic.co/elasticsearch/elasticsearch:7.6.1   "/usr/local/bin/do..."   5 minutes ago       Up 5 minutes        0.0.0.0:9200->9200/tcp, 9300/tcp   es01
        706171c7357b        docker.elastic.co/elasticsearch/elasticsearch:7.6.1   "/usr/local/bin/do..."   5 minutes ago       Up 5 minutes        9200/tcp, 9300/tcp                 es02
        941ebc831b93        docker.elastic.co/elasticsearch/elasticsearch:7.6.1   "/usr/local/bin/do..."   5 minutes ago       Up 5 minutes        9200/tcp, 9300/tcp                 es03
        
- **4 提交一个_cat/nodes 请求来查看节点是否被拉起来以及运行状态**

        curl -X GET "localhost:9200/_cat/nodes?v&pretty"

    **查看节点状态结果：**

        # curl -X GET "localhost:9200/_cat/nodes?v&pretty"
        ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
        172.18.0.4           22          86   8    0.16    0.58     0.70 dilm      -      es03
        172.18.0.3           16          86   7    0.16    0.58     0.70 dilm      *      es01
        172.18.0.2           27          86   7    0.16    0.58     0.70 dilm      -      es02

- **5 停止集群**

        #数据将会保留
        docker-compose down

        #数据将被清除
        docker-compose down -v

        #重启集群
        docker-compose up -d
        
## 产品中使用注意事项

- **1 设置 vm.max_map_count 至少为262144**

        grep vm.max_map_count /etc/sysctl.conf
        vm.max_map_count=262144

        #如果没有设置运行如下命令
        sysctl -w vm.max_map_count=262144

- **2 配置文件必须被elasticsearch用户可读**

    默认情况下，Elasticsearch使用uid：gid 1000：0作为用户elasticsearch在容器内运行。

    如果您要绑定安装本地目录或文件，则elasticsearch用户必须可以读取它。 此外，该用户必须对数据和日志目录具有写权限。

    例如：准备一个本地目录通过mount来存储数据

        mkdir esdatadir
        chmod g+rwx esdatadir
        chgrp 0 esdatadir
参考：[官方网站](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_pulling_the_image)