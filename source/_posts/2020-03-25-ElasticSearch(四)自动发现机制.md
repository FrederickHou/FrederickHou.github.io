---
layout:     post
title:      Elasticsearch(四)自动发现机制-Zen discovery
subtitle:   Elasticsearch 节点集群服务发现机制
date:       2020-03-25
author:     Frederick
header-img: img/es.png
categories:
    - 技术笔记
tags:
    - ElasticSearch
toc: true
cover: https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=2225621002,3316765983&fm=26&gp=0.jpg
---
# ElasticSearch(四)发现机制 - Zen discovery

    注：本篇文章是基于ElasticSearch V6 大版本为基础

## 发现方式
Elasticsearch的默认发现模块是Zen discovery。它提供了单播和基于文件的发现，可以通过插件扩展到支持云环境和其他形式的发现。

Zen Discovery 是与其他模块集成的，例如，节点之间的所有通信都使用 transport 模块完成。某个节点通过 发现机制 找到其他节点是使用 Ping 的方式实现的。

Zen Discovery 使用种子节点(seed nodes)列表来开始发现过程。在启动时，或者在选举新主节点的时候，Elasticsearch 会尝试连接到其列表中的每个种子节点，并与他们进行类似'闲聊'的对话，以查找其他节点并构建集群的完整成员图。

默认情况下，有两种方法可用于配置种子节点列表：单播和基于文件。建议种子节点列表主要由集群中那些 Master-eligible 的节点组成

    Master-eligible：node.master参数设置为 true（默认）的节点，使其有资格被选为控制集群的主节点

### 单播

单播发现 配置静态主机列表以用作种子节点。 可以将这些主机指定为 主机名 或 IP地址。 指定为主机名的主机在每轮 ping 操作期间解析为 IP 地址。 请注意，如果您处于 DNS 解析随时间变化的环境中，则可能需要调整 JVM安全设置。

可以在 elasticsearch.yml 配置文件中使用discovery.zen.ping.unicast.hosts参数静态设置设置主机列表。如下所示：

    discovery.zen.ping.unicast.hosts: ["node1", "node2"]

具体的值是一个主机数组或逗号分隔的字符串。每个值应采用host：port或host的形式（其中port默认为设置transport.profiles.default.port，如果未设置则返回transport.tcp.port）。 请注意，必须将IPv6主机置于括号内。 此设置的默认值为127.0.0.1，[:: 1]。

另外，discovery.zen.ping.unicast.resolve_timeout 配置在每轮ping操作中等待DNS查找的时间。需要指定时间单位，默认为5秒。

或者在Kubernetes平台中创建Headless Service 通过9300端口来发现待加入的种子节点。具体事例请参照：[Elasticsearch (三)容器化方案](https://www.frederickhou.com/2020/03/24/Elasticsearch-(%E4%B8%89)%E5%AE%B9%E5%99%A8%E5%8C%96%E6%96%B9%E6%A1%88/)

    discovery.zen.ping.unicast.hosts: es-cluster-discovery

### 基于文件
除了静态discovery.zen.ping.unicast.hosts 设置提供的主机之外，还可以通过外部文件提供主机列表。Elasticsearch在更改时会重新加载此文件，以便种子节点列表可以动态更改，而无需重新启动每个节点。例如，这为在Docker容器中运行的Elasticsearch实例提供了一种方便的机制，可以动态提供一个IP地址列表，以便在节点启动时无法知道这些IP地址时连接到Zen discovery。

要启用基于文件的发现，请文件按如下方式配置hosts提供程序

    discovery.zen.hosts_provider：file

该文件的格式是每行指定一个节点条目。每个节点条目由主机（主机名或IP地址）和可选的传输端口号组成。如果指定了端口号，必须在主机（在同一行）之后使用“：”分割。如果未指定端口号，则使用默认值9300。

例如，这是 unicast_hosts.txt 具有四个参与单播发现的节点的集群的示例，其中一些节点未在默认端口上运行：

    192.168.1.5
    192.168.1.6:9305
    192.168.1.7:10005
    # an IPv6 address
    [2001:0db8:85a3:0000:0000:8a2e:0370:7334]:9301

允许使用主机名而不是IP地址（类似于 discovery.zen.ping.unicast.hosts）。必须在括号中指定IPv6地址，并在括号后面添加端口。

## 主节点选举

作为 ping 过程的一部分，一个集群的主节点需要是被选举或者加入进来的(即选举主节点也会执行ping，其他的操作也会执行ping)。这个过程是自动执行的。通过配置discovery.zen.ping_timeout来控制节点加入某个集群或者开始选举的响应时间(默认3s)。

在这段时间内有3个 ping 会发出。如果超时,重新启动 ping 程序。在网络缓慢时，3秒时间可能不够，这种情况下，需要慎重增加超时时间，增加超时时间会减慢选举进程。

一旦节点决定加入一个存在的集群，它会发出一个加入请求给主节点，这个请求的超时时间由discovery.zen.join_time控制，默认是 ping 超时时间(discovery.zen.ping_timeout)的20倍。

当主节点停止或者出现问题，集群中的节点会重新 ping 并选举一个新节点。有时一个节点也许会错误的认为主节点已死，所以这种 ping 操作也可以作为部分网络故障的保护性措施。在这种情况下，节点将只从其他节点监听有关当前活动主节点的信息。

如果discovery.zen.master_election.ignore_non_master_pings设置为true时（默认值为false），node.master为false的节点不参加主节点的选举，同时选票也不包含这种节点。

通过设置node.master为false，可以将节点设置为非备选主节点，永远没有机会成为主节点。

discovery.zen.minimum_master_nodes设置了最少有多少个备选主节点参加选举，同时也设置了一个主节点需要控制最少多少个备选主节点才能继续保持主节点身份。如果控制的备选主节点少于discovery.zen.minimum_master_nodes个，那么当前主节点下台，重新开始选举。

discovery.zen.minimum_master_nodes必须设置一个恰当的备选主节点值(quonum，一般设置 为备选主节点数/2+1)，尽量避免只有两个备选主节点，因为两个备选主节点quonum应该为2，那么如果一个节点出现问题，另一个节点的同意人数最多只能为1，永远也不能选举出新的主节点，这时就发生了**脑裂**现象。

## 脑分裂（split-brain)

假设拥有一个10个节点组成的集群，有3个节点从集群中 断开连接，由于发现机制，这3个节点可能会组成一个新的集群，这样就产生了两个同名的集群，这就是脑分裂（split-brain)。 为了避免这种分裂的出现，可以设置以下属性：

    discovery.zen.minium_master_nodes:n/2+1
