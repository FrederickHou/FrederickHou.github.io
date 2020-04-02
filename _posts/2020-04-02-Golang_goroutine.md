---
layout:     post
title:      Golang goroutine讲解
subtitle:   
date:       2020-04-02
author:     Frederick
header-img: img/go.jpeg
catalog: true
tags:
    - Golang
---

## goroutine简介
goroutine是go语言中最为NB的设计，也是其魅力所在，goroutine的本质是协程，是实现并行计算的核心。goroutine使用方式非常的简单，只需使用go关键字即可启动一个协程，并且它是处于异步方式运行，你不需要等它运行完成以后在执行以后的代码。GO默认是使用一个CPU核的，除非设置runtime.GOMAXPROCS。

    go func()//通过go关键字启动一个协程来运行函数   

## 概念普及
- **并发**：
**一个cpu上**能同时执行多项任务，在很短时间内，cpu来回切换任务执行(在某段很短时间内执行程序a，然后又迅速得切换到程序b去执行)，有时间上的重叠（宏观上是同时的，微观仍是顺序执行）,这样看起来多个任务像是同时执行，这就是并发。
- **并行**
当系统有**多个CPU**(多核)时,每个CPU同一时刻都运行任务，互不抢占自己所在的CPU资源，同时进行，称为并行。
- **进程**
cpu在切换程序的时候，如果不保存上一个程序的状态（也就是我们常说的context--上下文），直接切换下一个程序，就会丢失上一个程序的一系列状态，于是引入了进程这个概念，用以划分好程序运行时所需要的资源。因此进程就是一个程序运行时候的所需要的基本资源单位（也可以说是程序运行的一个实体）。(系统进行资源分配和调度的基本单位)
- **线程**
cpu切换多个进程的时候，会花费不少的时间，因为切换进程需要切换到内核态，而每次调度需要内核态都需要读取用户态的数据，进程一旦多起来，cpu调度会消耗一大堆资源，因此引入了线程的概念，线程本身几乎不占有资源，他们共享进程里的资源，内核调度起来不会那么像进程切换那么耗费资源。
- **协程**
协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈。因此，协程能保留上一次调用时的状态（即所有局部状态的一个特定组合），每次过程重入时，就相当于进入上一次调用的状态，换种说法：进入上一次离开时所处逻辑流的位置。线程和进程的操作是由程序触发系统接口，最后的执行者是系统；协程的操作执行者则是用户自身程序，goroutine也是协程。协程位于线程级别

## 调度模型简介
groutine能拥有强大的并发实现是通过GPM调度模型实现。
Go的调度器内部有四个重要的结构：M，P，G，Sched
- **M**:代表内核级线程，一个M就是一个线程，goroutine就是跑在M之上的；M是一个很大的结构，里面维护小对象内存cache（mcache）、当前执行的goroutine、随机数发生器等等非常多的信息。
- **G**:代表一个goroutine，它有自己的栈，instruction pointer和其他信息（正在等待的channel等等），用于调度。
- **P**:全称是Processor，处理器，它的主要用途就是用来执行goroutine的，所以它也维护了一个goroutine队列，里面存储了所有需要它来执行的goroutine。
- **Sched**：代表调度器，它维护有存储M和G的队列以及调度器的一些状态信息等。

## 调度实现场景

**一般调度如下图所示：**
![goroutine.png](https://upload-images.jianshu.io/upload_images/17904159-cb5bc6d48c41a052.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

从上图中看，有2个物理线程M1和M2，每一个M都拥有一个处理器P，每一个也都有一个正在运行的goroutine。
P的数量可以通过GOMAXPROCS()来设置，它其实也就代表了真正的并发度，即有多少个goroutine可以同时运行。
图中灰色的那些goroutine并没有运行，而是出于ready的就绪态，正在等待被调度。P维护着这个队列（称之为runqueue），
Go语言里，启动一个goroutine很容易：go function 就行，所以每有一个go语句被执行，runqueue队列就在其末尾加入一个
goroutine，在下一个调度点，就从runqueue中取出（如何决定取哪个goroutine？）一个goroutine执行。

**当一个OS线程M1陷入阻塞时：**

P转而在运行M2，图中的M2可能是正被创建，或者从线程缓存中取出。当M1返回时，它必须尝试取得一个P来运行goroutine，一般情况下，它会从其他的OS线程那里拿一个P过来，

如果没有拿到的话，它就把goroutine放在一个global runqueue里，然后自己睡眠（放入线程缓存里）。所有的P也会周期性的检查global runqueue并运行其中的goroutine，否则global runqueue上的goroutine永远无法执行。

**调度分配不均：如下图所示**

![goroutine1.png](https://upload-images.jianshu.io/upload_images/17904159-5ee5fd44bfd36c3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

P所分配的任务G很快就执行完了（分配不均），这就导致了这个处理器P很忙，但是其他的P还有任务，此时如果global runqueue没有任务G了，那么P不得不从其他的P里拿一些G来执行。一般来说，如果P从其他的P那里要拿任务的话，**一般就拿run queue的一半**，这就确保了每个OS线程都能充分的使用。

## 使用goroutine

### goroutine异常捕获

    package main

    import (
        "fmt"
        "time"
    )

    func addele(a []int ,i int)  {
        defer func() {    //匿名函数捕获错误
            err := recover()
            if err != nil {
                fmt.Println(err)
            }
        }()
    a[i]=i
    fmt.Println(a)
    }

    func main() {
        Arry := make([]int,4)
        for i :=0 ; i<10 ;i++{
            go addele(Arry,i)
        }
        time.Sleep(time.Second * 2)
    }
    //结果

### 同步的goroutine
由于goroutine是异步执行的，那很有可能出现主程序退出时还有goroutine没有执行完，此时goroutine也会跟着退出。此时如果想等到所有goroutine任务执行完毕才退出，go提供了sync包和channel来解决同步问题，当然如果你能预测每个goroutine执行的时间，你还可以通过time.Sleep方式等待所有的groutine执行完成以后在退出程序(如上面的列子)。

**示例一：使用sync包同步goroutine**
sync大致实现方式:
WaitGroup 等待一组goroutinue执行完毕. 主程序调用 Add 添加等待的goroutinue数量. 每个goroutinue在执行结束时调用 Done ，此时等待队列数量减1.，主程序通过Wait阻塞，直到等待队列为0.

    package main

    import (
        "fmt"
        "sync"
    )

    func cal(a int , b int ,n *sync.WaitGroup)  {
        c := a+b
        fmt.Printf("%d + %d = %d\n",a,b,c)
        defer n.Done() //goroutinue完成后, WaitGroup的计数-1

    }

    func main() {
        var go_sync sync.WaitGroup //声明一个WaitGroup变量
        for i :=0 ; i<10 ;i++{
            go_sync.Add(1) // WaitGroup的计数加1
            go cal(i,i+1,&go_sync)  
        }
        go_sync.Wait()  //等待所有goroutine执行完毕
    }

**示例二：通过channel实现goroutine之间的同步。**
channel实现方式：
通过channel能在多个groutine之间通讯，当一个goroutine完成时候向channel发送退出信号,等所有goroutine退出时候，利用for循环channel,取channel中的信号，若取不到数据便会阻塞原理，等待所有goroutine执行完毕，使用该方法有个前提是你已经知道了你启动了多少个goroutine。

    package main

    import (
        "fmt"
        "time"
    )

    func cal(a int , b int ,Exitchan chan bool)  {
        c := a+b
        fmt.Printf("%d + %d = %d\n",a,b,c)
        time.Sleep(time.Second*2)
        Exitchan <- true
    }

    func main() {

        Exitchan := make(chan bool,10)  //声明并分配管道内存
        for i :=0 ; i<10 ;i++{
            go cal(i,i+1,Exitchan)
        }
        for j :=0; j<10; j++{   
            <- Exitchan  //取信号数据，如果取不到则会阻塞
        }
        close(Exitchan) // 关闭管道
    }

### goroutine之间的通讯
goroutine本质上是协程，可以理解为不受内核调度，而受go调度器管理的线程。goroutine之间可以通过channel进行通信或者说是数据共享，当然你也可以使用全局变量来进行数据共享。
示例代码：采用生产者和消费者模式

    package main

    import (
        "fmt"
        "sync"
    )

    func Productor(mychan chan int,data int,wait *sync.WaitGroup)  {
        mychan <- data
        fmt.Println("product data：",data)
        defer wait.Done()
    }
    func Consumer(mychan chan int,wait *sync.WaitGroup)  {
        a := <- mychan
        fmt.Println("consumer data：",a)
        defer wait.Done()
    }
    func main() {

        datachan := make(chan int, 100)   //通讯数据管道
        var wg sync.WaitGroup

        for i := 0; i < 10; i++ {
            go Productor(datachan, i,&wg) //生产数据
            wg.Add(1)
        }
        for j := 0; j < 10; j++ {
            go Consumer(datachan,&wg)  //消费数据
            wg.Add(1)
        }
        wg.Wait()
    }