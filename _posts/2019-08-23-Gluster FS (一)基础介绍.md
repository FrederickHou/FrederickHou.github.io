---
layout:     post
title:      Gluster Fs(一)基础知识介绍
subtitle:  带你认识Gluster fs 分布式文件系统
date:       2019-08-19
author:     Frederick
header-img: img/ldap2.jpg
catalog: true
tags:
    - 大数据
    - 分布式
---


<iframe src="//music.163.com/outchain/player?type=2&id=1359356908&auto=1&height=66" width=300 height=86></iframe>


## 基础知识铺垫(什么是分布式文件系统？)

分布式文件系统（Distributed File System）是指文件系统管理的物理存储资源并不直接与本地节点相连，而是分布于计算网络中的一个或者多个节点的计算机上。目前意义上的分布式文件系统大多都是由多个节点计算机构成，结构上是典型的客户机/服务器模式。流行的模式是当客户机需要存储数据时，服务器指引其将数据分散的存储到多个存储节点上，以提供更快的速度，更大的容量及更好的冗余特性。

    目前流行的分布式文件系统有许多，如MooseFS、OpenAFS、GoogleFS等。

## GlusterFS 简介

GlusterFS系统是一个可扩展的网络文件系统，相比其他分布式文件系统，GlusterFS具有高扩展性、高可用性、高性能、可横向扩展等特点，并且其没有元数据服务器的设计，让整个服务没有单点故障的隐患。

### 关键词介绍

- **Brick:** GFS中的存储单元，通过是一个受信存储池中的服务器的一个导出目录。可以通过主机名和目录名来标识，如'HOME:PATH'
- **Client:** 挂载了GFS卷的设备
- **Extended Attributes:** xattr是一个文件系统的特性，其支持用户或程序关联文件/目录和元数据。
- **FUSE:** Filesystem Userspace是一个可加载的内核模块，其支持非特权用户创建自己的文件系统而不需要修改内核代码。通过在用户空间运行文件系统的代码通过FUSE代码与内核进行桥接。
- **Geo-Replication：** 跨地理位置的副本
- **GFID:** GFS卷中的每个文件或目录都有一个唯一的128位的数据相关联，其用于模拟inode
- **Namespace:** 每个Gluster卷都导出单个ns作为POSIX的挂载点
- **Node:** 一个拥有若干brick的设备
- **RDMA:** 远程直接内存访问，支持不通过双方的OS进行直接内存访问。
- **RRDNS:** round robin DNS是一种通过DNS轮转返回不同的设备以进行负载均衡的方法
- **Self-heal:** 用于后台运行检测复本卷中文件和目录的不一致性并解决这些不一致。
- **Split-brain:** 脑裂
- **Volfile:** glusterfs进程的配置文件，通常位于/var/lib/glusterd/vols/volname
- **Volume:** 一组bricks的逻辑集合

### 无元数据设计

元数据是用来描述一个文件或给定区块在分布式文件系统中所在的位置，简而言之就是某个文件或某个区块存储的位置。传统分布式文件系统大都会设置元数据服务器或者功能相近的管理服务器，主要作用就是用来管理文件与数据区块之间的存储位置关系。相较其他分布式文件系统而言，GlusterFS并没有集中或者分布式的元数据的概念，取而代之的是弹性哈希算法。集群中的任何服务器和客户端都可以利用哈希算法、路径及文件名进行计算，就可以对数据进行定位，并执行读写访问操作。

这种设计带来的好处是极大的提高了扩展性，同时也提高了系统的性能和可靠性；另一显著的特点是如果给定确定的文件名，查找文件位置会非常快。但是如果要列出文件或者目录，性能会大幅下降，因为列出文件或者目录时，需要查询所在节点并对各节点中的信息进行聚合。此时有元数据服务的分布式文件系统的查询效率反而会提高许多。

## 应用

主要应用在集群系统中，具有很好的可扩展性。软件的结构设计良好，易于扩展和配置，通过各个模块的灵活搭配以得到针对性的解决方案。可解决以下问题：网络存储，联合存储(融合多个节点上的存储空间)，冗余备份，大文件的负载均衡(分块)。由于缺乏一些关键特性，可靠性也未经过长时间考验，还不适合应用于需要提供 24 小时不间断服务的产品环境。目前适合应用于大数据量的离线应用。
　　由于它良好的软件设计，以及由专门的公司负责开发，进展非常迅速，几个月或者一年后将会有很大的改进，非常值得期待。
GlusterFS通过Infiniband RDMA 或者Tcp/Ip 方式将许多廉价的x86 主机，通过网络互联成一个并行的网络文件系统

### Gluster Fs 集群模式卷的应用介绍

#### 1. 分布式卷（Distributed Volume）

又称其为哈希卷，近似于RAID0，文件没有分片，文件根据hash算法写入各个节点的硬盘上，优点是容量大，缺点是没数据冗余即可靠性无法保障。

**创建分布式卷指令如下：**

    gluster volume create test-distributed-volume server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4
    
#### 2. 复制卷（Replicated Volume）

复制卷设定复制的份数，决定集群的大小，通常与分布式卷或者条带卷组合使用，解决前两种存储卷的冗余缺陷。缺点是磁盘利用率低。

复本卷在创建时可指定复本的数量，通常为2或者3，复本在存储时会在卷的不同brick上，因此有几个复本就必须提供至少多个brick，当其中一台服务器岩机后，可以从另一台服务器读取数据，因此复制卷提高了数据可靠性的同时，还提供了数据冗余的功能。

**创建分布式卷指令如下：**

    gluster volume create test-replicated-volume replica 2 transport tcp server1:/exp1 server2:/exp2

#### 3. 分布式复制卷（Distributed Replicated Volume）

分布式复制GlusterFS卷结合了分布式和复制Gluster卷的特点。

**创建分布式卷指令如下：**

    gluster volume create test-distributed-replicated-volume  replica 2 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4

#### 3. 条带卷（Striped Volume）

文件是分片均匀写在各个节点的硬盘上的，优点是分布式读写，性能整体较好。缺点是没冗余，分片随机读写可能会导致硬盘IOPS饱和。

**创建分布式卷指令如下：**

    gluster volume create test-striped-volume stripe 2 transport tcp server1:/exp1 server2:/exp2

#### 4. 分布式条带卷（Distributed Striped Volume）

当单个文件的体型十分巨大，客户端数量更多时，条带卷已经无法满足需求，此时将分布式与条带化结合起来是一个比较好的选择。其性能与服务器数量有关。

**创建分布式卷指令如下：**

    gluster volume create test-distributed-striped-volume stripe 4 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4 server5:/exp5 server6:/exp6 server7:/exp7 server8:/exp8


**博客著作权归本作者所有，任何形式的转载都请联系作者获得授权并注明出处。**
