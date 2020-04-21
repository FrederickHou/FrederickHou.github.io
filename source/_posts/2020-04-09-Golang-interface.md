---
layout:     post
title:      Golang interface
subtitle:   
date:       2020-04-09
author:     Frederick
header-img: img/go.jpg
categories:
    - 技术笔记
    - Golang 笔记
tags:
    - Golang
toc: true
---

# Golang interface 

## 接口概念

**接口** 即一组方法定义的集合，定义了对象的一组行为，由具体的类型实例实现具体的方法。换句话说，一个接口就是定义（规范或约束），而方法就是实现，接口的作用应该是将定义与实现分离，降低耦合度。习惯用“er”结尾来命名，例如“Reader”。接口与对象的关系是多对多，即一个对象可以实现多个接口，一个接口也可以被多个对象实现。

**​接口(interface)** 是Go语言整个类型系统的基石，其他语言的接口是不同组件之间的契约的存在，对契约的实现是强制性的，必须显式声明实现了该接口，这类接口称之为“侵入式接口”。而Go语言的接口是隐式存在，只要实现了该接口的所有函数则代表已经实现了该接口，并不需要显式的接口声明。

## 接口的作用

​接口是实现语言的多态性，**多态性（polymorphisn）** 是允许你将父对象设置成为和一个或更多的他的子对象相等的技术，赋值之后，父对象就可以根据当前赋值给它的子对象的特性以不同的方式运作。

简而言之，就是允许将子类类型的指针赋值给父类类型的指针。

即一个引用变量倒底会指向哪个类的实例对象，该引用变量发出的方法调用到底是哪个类中实现的方法，必须在由程序运行期间才能决定。不修改程序代码就可以改变程序运行时所绑定的具体代码，让程序可以选择多个运行状态，这就是多态性。多态分为编译时多态（静态多态）和运行时多态（动态多态），编译时多态一般通过方法重载实现，运行时多态一般通过方法重写实现。

## 接口示例

**非侵入式接口：** 一个类只需要实现了接口要求的所有函数就表示实现了该接口，并不需要显式声明

    package main
    import (
        "fmt"
    )
    // 接口1
    type File interface{
        Read(buf []byte)error
        Write(buf []byte)error
        Close()error
    }
    // 接口2
    type Receiver interface{
        Close()error
    }

    type Note struct{

    }
    func (* Note)Read(buf []byte)error{
        fmt.Println("Note Read")
        return nil
    }

    func (* Note)Write(buf []byte)error{
        fmt.Println("Note Write")
        return nil
    }

    func (*Note)Close()error{
        fmt.Println("Note Close")
        return nil
    }
    func main() {
        //接口赋值,Note类实现了File和Receiver接口，即接口所包含的所有方法
        var file File = new(Note)
        var recv Receiver = new(Note)
        file.Close()
        recv.Close()
    }

### 接口赋值给另一个接口

- **1.** 只要两个接口拥有相同的方法列表（与次序无关），即是两个相同的接口，可以相互赋值
- **2.** 接口赋值只需要接口A的方法列表是接口B的子集（即假设接口A中定义的所有方法，都在接口B中有定义），那么B接口的实例可以赋值给A的对象。反之不成立，即子接口B包含了父接口A，因此可以将子接口的实例赋值给父接口。
即子接口实例实现了子接口的所有方法，而父接口的方法列表是子接口的子集，则子接口实例自然实现了父接口的所有方法，因此可以将子接口实例赋值给父接口。

        package main
        import (
            "fmt"
        )

        type Writer interface{    //父接口 父接口是子接口的子集
            Write(buf []byte)error
        }
        type ReadWriter interface{    //子接口
            Read(buf []byte)error
            Write(buf []byte)error
        }

        type Note struct{

        }
        func (* Note)Read(buf []byte)error{
            fmt.Println("Note Read")
            return nil
        }

        func (* Note)Write(buf []byte)error{
            fmt.Println("Note Write")
            return nil
        }

        func main() {
            var file1 ReadWriter=new(Note)   //子接口实例
            var file2 Writer=file1 
            file1.Write(nil) 
            file2.Write(nil)
        }

### 接口组合

    //接口组合类似类型组合，只不过只包含方法，不包含成员变量

    package main
    import (
        "fmt"
    )

    type Receiver interface{    //父接口 父接口是子接口的子集
        Close(buf []byte)error
    }
    type ReadWriter interface{    //子接口
        Read(buf []byte)error
        Write(buf []byte)error
    }

    type File interface{
        Receiver
        ReadWriter
    }

    type Note struct{

    }
    func (* Note)Read(buf []byte)error{
        fmt.Println("Note Read")
        return nil
    }

    func (* Note)Write(buf []byte)error{
        fmt.Println("Note Write")
        return nil
    }

    func (* Note)Close(buf []byte)error{
        fmt.Println("Note Close")
        return nil
    }

    func main() {
        var file1 File=new(Note)   //子接口实例
        file1.Write(nil) 
    }