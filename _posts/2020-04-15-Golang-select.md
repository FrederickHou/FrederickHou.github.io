---
layout:     post
title:      Golang select介绍
subtitle:   
date:       2020-04-15
author:     Frederick
header-img: img/go.jpg
catalog: true
tags:
    - Golang
---

# Golang select

Go 语言中的 select 关键字也能够让 Goroutine 同时等待多个 Channel 的可读或者可写，在多个文件或者 Channel 发生状态改变之前，select 会一直阻塞当前线程或者 Goroutine。

select 是一种与 switch 相似的控制结构，与 switch 不同的是，select 中虽然也有多个 case，但是这些 case 中的表达式必须都是 Channel 的收发操作。下面的代码就展示了一个包含 Channel 收发操作的 select 结构：