---
layout:     post
title:      共享内存的使用实现原理
subtitle:   ""
date:       2020-04-09
author:     Frederick
header-img: img/post-bg-os-metro.jpg
categories:
    - 技术笔记
    - 操作系统
tags:
    - 效率
    - 操作系统
toc: true
---

# 享内存的使用实现原理

共享内存可以说是最有用的进程间通信方式，也是最快的IPC形式。两个不同进程A、B共享内存的意思是，同一块物理内存被映射到进程A、B各自的进程地址空间。进程A可以即时看到进程B对共享内存中数据的更新，反之亦然。由于多个进程共享同一块内存区域，必然需要某种同步机制，互斥锁和信号量都可以。使用共享内存时需要注意多个进程对共享内存的同步访问。


### 共享内存段最大限制

SHMMNI为128，表示系统中最多可以有128个共享内存对象。

## ELF 是什么 ？

其大小与程序中全局变量的是否初始化有什么关系（注意未初始化的数据放在 bss 段）

**Linux ELF**  
ELF = Executable and Linkable Format ，可执行连接格式