---
layout:     post
title:      Golang 异常处理
subtitle:   
date:       2020-04-03
author:     Frederick
header-img: img/go.jpg
catalog: true
tags:
    - Golang
---


# Golang 异常处理

## 简介
Go语言追求简洁优雅，所以，Go语言不支持传统的 try…catch…finally 这种异常，因为Go语言的设计者们认为，将异常与控制结构混在一起会很容易使得代码变得混乱。因为开发者很容易滥用异常，甚至一个小小的错误都抛出一个异常。在Go语言中，使用多值返回来返回错误。不要用异常代替错误，更不要用来控制流程。在极个别的情况下，也就是说，遇到真正的异常的情况下（比如除数为0了）。才使用Go中引入的Exception处理：defer, panic, recover。

## defer函数
### 介绍

defer语句是Go中一个非常有用的特性，可以将一个方法延迟到包裹该方法的方法返回时执行，在实际应用中，defer语句可以充当其他语言中try…catch…的角色，也可以用来处理关闭文件句柄等收尾操作。

 **有以下特点：**

- **1.** defer在声明时不会执行，而是推迟执行，在return执行前，**倒序执行defer[先进后出]**，一般用于释放资源，清理数据，记录日志，异常处理等。

- **2.** defer有一个特性：**即使函数抛出异常，defer仍会被执行**，这样不会出现程序错误导致资源不被释放，或者因为第三方包的异常导致程序崩溃。

### defer函数示例

    package main

    import "fmt"

    func stackingDefers() {
        defer func() {
            fmt.Println("1")
        }()
        defer func() {
            fmt.Println("2")
        }()
        defer func() {
            fmt.Println("3")
        }()
    }
    func main() {
        stackingDefers()
    }

**输出结果：**

    3
    2
    1

**注释：** 根据输出结果发现：当一个方法中有多个defer时， defer会将要延迟执行的方法“压栈”，当defer被触发时，将所有“压栈”的方法“出栈”并执行。所以defer的执行顺序是LIFO的。


### defer 在匿名函数中的坑

    package main

    import "fmt"

    func returnValues() int {
        var result int
        defer func() {
            result++
            fmt.Println("defer")
        }()
        return result
    }

    func namedReturnValues() (result int) {
        defer func() {
            result++
            fmt.Println("defer")
        }()
        return result
    }

    func main() {
        fmt.Println(returnValues())
        fmt.Println(namedReturnValues())
    }

**输出结果：**

    defer
    0
    defer
    1

**注释:** 上面的方法看到输出0，下面的方法输出1。上面的方法使用了匿名返回值，下面的使用了命名返回值，除此之外其他的逻辑均相同，为什么输出的结果会有区别呢？

要搞清这个问题首先需要了解defer的执行逻辑，文档中说defer语句在方法返回“时”触发，也就是说return和defer是“同时”执行的。以匿名返回值方法举例，过程如下。

将result赋值给返回值（可以理解成Go自动创建了一个返回值retValue，相当于执行retValue = result）
然后检查是否有defer，如果有则执行
返回刚才创建的返回值（retValue）
在这种情况下，defer中的修改是对result执行的，而不是retValue，所以defer返回的依然是retValue。在命名返回值方法中，由于返回值在方法定义时已经被定义，所以没有创建retValue的过程，result就是retValue，defer对于result的修改也会被直接返回。

### 判断执行没有err之后，再defer释放资源

    resp, err := http.Get(url)
    // 先判断操作是否成功
    if err != nil {
        return err
    }
    // 如果操作成功，再进行Close操作
    defer resp.Body.Close()

**注释：** 一些获取资源的操作可能会返回err参数，我们可以选择忽略返回的err参数，但是如果要使用defer进行延迟释放的的话，需要在使用defer之前先判断是否存在err，如果资源没有获取成功，即没有必要也不应该再对资源执行释放操作。如果不判断获取资源是否成功就执行释放操作的话，还有可能导致释放方法执行错误。

### 调用os.Exit时defer不会被执行

    package main

    import (
        "fmt"
        "os")

    func deferExit() {
        defer func() {
            fmt.Println("defer")
        }()
        fmt.Println("exit")
        os.Exit(0)
    }

    func main() {
        deferExit()
    }
**结果输出：**

    exit

**注释:** 当发生panic时，所在goroutine的所有defer会被执行，但是当调用os.Exit()方法退出程序时，defer并不会被执行。

## panic()和recover()函数
Golang 语言内置函数panic()和recover()来处理程序中的错误。

    func recover() interface{}
    func panic(interface{})

### panic()函数

当函数执行触发了panic()函数时，如果没有使用到defer关键字，函数执行流程会被立即终止；如果使用了defer关键字，则会逐层执行defer语句，直到所有的函数被终止。

错误信息，包括panic函数传入的参数，将会被报告出来。

    package main


    func raiseErr(err interface{}){
        panic(err)
    }

    func main() {
        raiseErr("Unknow err")
    }

**输出结果：**

    panic: Unknow err

    goroutine 1 [running]:
    main.raiseErr(...)
            E:/coding/go/test/test.go:5
    main.main()
            E:/coding/go/test/test.go:9 +0x40

**注释:** 以上程序直接终止抛出异常。、相当于C++/Python里面的异常抛出，如果外围函数没有接住异常，便使得程序终止退出。

### recover()函数

recover()函数用于终止错误处理流程。一般使用在defer关键字后以有效截取错误处理流程。如果没有在发生异常的goroutine中明确调用恢复过程（使用recover关键字） ，会导致该goroutine所属的进程打印异常信息后直接退出。

    package main
    import (
        "fmt"
    )

    func raiseErr(err interface{}){
        panic(err)
    }

    func main() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Printf("Exception error caught: %v\n", r)
            }
        }()
        raiseErr("Unknow err")
    }
**结果输出：**

    Exception error caught: Unknow err

**注释:** raiseErr("Unknow err"))中触发了错误处理流程，该匿名defer函数都将在函数退出时得到执行。 recover()函数执行将使得该错误处理过程终止。此时错误处理流程被触发时，程序传给panic函数的参数不为nil，则该函数还会打印详细的错误信息。