---
layout:     post
title:      Golang channel
subtitle:   
date:       2020-04-02
author:     Frederick
header-img: img/bk_go.jpg
catalog: true
tags:
    - Golang
---

## channel简介

channel俗称管道，用于数据传递或数据共享，其本质是一个先进先出的队列，使用goroutine+channel进行数据通讯简单高效，同时也线程安全，多个goroutine可同时修改一个channel，**不需要加锁** 。

channel可分为三种类型：

- 只读channel：只能读channel里面数据，不可写入

- 只写channel：只能写数据，不可读

- 一般channel：可读可写

### channel使用

定义和声明

    var readOnlyChan <-chan int            // 只读chan
    var writeOnlyChan chan<- int           // 只写chan
    var mychan  chan int                     //读写channel
    //定义完成以后需要make来分配内存空间，不然使用会deadlock
    mychannel = make(chan int,10)

    //或者
    read_only := make (<-chan int,10)//定义只读的channel
    write_only := make (chan<- int,10)//定义只写的channel
    read_write := make (chan int,10)//可同时读写
    
    //操作
    write_only <- "wd"  //写数据
    a := <- read_only //读取数据
    a, ok := <- read_only  //优雅的读取数据

**注：**

- 读写操作注意：
  - 管道如果未关闭，在读取超时会则会引发deadlock异常
  - 管道如果关闭进行写入数据会pannic
  - 当管道中没有数据时候再行读取或读取到默认值，如int类型默认值是0
- 循环管道注意：
  - 使用range循环管道，如果管道未关闭会引发deadlock错误。
  - 如果采用for死循环已经关闭的管道，当管道没有数据时候，读取的数据会是管道的默认值，并且循环不会退出。

示例代码：

    package main

    import (
        "fmt"
        "time"
    )

    func main() {
        mychannel := make(chan int,10)
        for i := 0;i < 10;i++{
            mychannel <- i
        }
        close(mychannel)  //关闭管道
        fmt.Println("data lenght: ",len(mychannel))
        for  v := range mychannel {  //循环管道
            fmt.Println(v)
        }
        fmt.Printf("data lenght:  %d",len(mychannel))
    }

### 带缓冲区channel和不带缓冲区channel

带缓冲区channel：定义声明时候制定了缓冲区大小(长度)，可以保存多个数据。

    ch := make(chan int ,10) //带缓冲区

不带缓冲区channel：只能存一个数据，并且只有当该数据被取出时候才能存下一个数据。

    ch := make(chan int) //不带缓冲区

### channel实现作业池

创建三个channel，一个channel用于接受任务，一个channel用于保持结果，还有个channel用于决定程序退出的时候。

    package main

    import (
        "fmt"
    )

    func Task(taskch, resch chan int, exitch chan bool) {
        defer func() {   //异常处理
            err := recover()
            if err != nil {
                fmt.Println("do task error：", err)
                return
            }
        }()

        for t := range taskch { //  处理任务
            fmt.Println("do task :", t)
            resch <- t //
        }
        exitch <- true //处理完发送退出信号
    }

    func main() {
        taskch := make(chan int, 20) //任务管道
        resch := make(chan int, 20)  //结果管道
        exitch := make(chan bool, 5) //退出管道
        go func() {
            for i := 0; i < 10; i++ {
                taskch <- i
            }
            close(taskch)
        }()

        for i := 0; i < 5; i++ {  //启动5个goroutine做任务
            go Task(taskch, resch, exitch)
        }

        go func() { //等5个goroutine结束
            for i := 0; i < 5; i++ {
                <-exitch
            }
            close(resch)  //任务处理完成关闭结果管道，不然range报错
            close(exitch)  //关闭退出管道
        }()

        for res := range resch{  //打印结果
            fmt.Println("task res：",res)
        }
    }

### select-case实现非阻塞channel

原理通过select+case加入一组管道，当满足（这里说的满足意思是有数据可读或者可写)select中的某个case时候，那么该case返回，若都不满足case，则走default分支。

    package main

    import (
        "fmt"
    )

    func send(c chan int)  {
        for i :=1 ; i<10 ;i++  {
        c <-i
        fmt.Println("send data : ",i)
        }
    }

    func main() {
        resch := make(chan int,20)
        strch := make(chan string,10)
        go send(resch)
        strch <- "wd"
        select {
        case a := <-resch:
            fmt.Println("get data : ", a)
        case b := <-strch:
            fmt.Println("get data : ", b)
        default:
            fmt.Println("no channel actvie")

        }
    }

### channel频率控制

在对channel进行读写的时，go还提供了非常人性化的操作，那就是对读写的频率控制，通过time.Ticke实现

    package main

    import (
        "time"
        "fmt"
    )

    func main(){
        requests:= make(chan int ,5)
        for i:=1;i<5;i++{
            requests<-i
        }
        close(requests)
        limiter := time.Tick(time.Second*1)
        for req:=range requests{
            <-limiter
            fmt.Println("requets",req,time.Now()) //执行到这里，需要隔1秒才继续往下执行，time.Tick(timer)上面已定义
        }
    }
