---
layout:     post
title:      Golang 闭包
subtitle:   
date:       2020-04-03
author:     Frederick
header-img: img/Go.png
catalog: true
tags:
    - Golang
---

# 闭包函数

## 匿名函数

匿名函数：不带函数名的函数，可以像变量一样被传递

    func(a,b int,z float32) bool{  //没有函数名
    return a*b<int(z)
    }
    f:=func(x,y int) int{
    return x+y
    }

## 闭包函数介绍

闭包是可以包含自由变量（未绑定到特定的对象）的代码块，这些变量不在代码块内或全局上下文中定义，而在定义代码块的环境中定义。要执行的代码块为自由变量提供绑定的计算环境（作用域）。闭包的价值在于可以作为一个变量对象来进行传递和返回。即可以把函数本身看作是一个变量，Go的匿名函数是一个闭包，Go闭包常用在go和defer关键字中。

## 闭包函数的坑

在for range中goroutine的方式使用闭包，如果没有给匿名函数传入一个变量，或新建一个变量存储迭代的变量，那么goroutine执行的结果会是最后一个迭代变量的结果，而不是每个迭代变量的结果。这是因为如果没有通过一个变量来拷贝迭代变量，那么闭包因为绑定了变量，当每个groutine运行时，迭代变量可能被更改。

**示例如下：**

- 问题case:

        package main

        import (
            "fmt"
            "sync"
        )
        func main() {
            var go_sync sync.WaitGroup
            values := []int{1,2,3}
            for _, val := range values {
                go func() {
                    fmt.Println(val)
                    defer go_sync.Done()
                }()
                go_sync.Add(1)
            }
            go_sync.Wait()

        }

    **结果输出：**

        3
        3
        3

- 规范case:

        package main

        import (
            "fmt"
            "sync"
        )
        func main() {
            var go_sync sync.WaitGroup
            values := []int{1,2,3}
            for _, val := range values {
                go func(val interface{}) {
                    fmt.Println(val)
                    defer go_sync.Done()
                }(val)
                go_sync.Add(1)
            }
            go_sync.Wait()

        }

    **结果输出：**

        3
        1
        2

## 闭包函数应用示例

    package main

    import "fmt"

    func adder() func(int) int {
        sum := 0
        return func(x int) int {
            sum += x
            return sum
        }
    }

    func main() {
        pos, neg := adder(), adder()
        for i := 0; i < 10; i++ {
            fmt.Println(pos(i),neg(-2*i),
            )
        }
    }

**结果输出：**

    0 0
    1 -2
    3 -6
    6 -12
    10 -20
    15 -30
    21 -42
    28 -56
    36 -72
    45 -90
    
## 总结

闭包并不是一门编程语言不可缺少的功能，但闭包的表现形式一般是以匿名函数的方式出现，就象上面说到的，能够动态灵活的创建以及传递，体现出函数式编程的特点。所以在一些场合，我们就多了一种编码方式的选择，适当的使用闭包可以使得我们的代码简洁高效。

## 注意点

由于闭包会使得函数中的变量都被保存在内存中，内存消耗很大，所以不能滥用闭包