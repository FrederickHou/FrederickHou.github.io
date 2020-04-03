---
layout:     post
title:      Golang 不定参数
subtitle:   
date:       2020-04-03
author:     Frederick
header-img: img/bk_go.jpg
catalog: true
tags:
    - Golang
---

# 不定参

可以把不定参数理解为一个数组切片，你可以自己组织一个数组切片，然后将其作为不定参数传给一个可以接受不定参数的函数,而且只能放在参数中的最后面。
特殊字符标志：**“...”**。
## 不定参数函数



    package main

    import (
        "fmt"
    )


    //不定参数函数
    func Add(a int, args ...int) (result int) {
        result += a
        for _, arg := range args {
            result += arg
        }
        return
    }

    func main() {
        fmt.Println(Add(1, 2, 3, 4, 5, 6, 7, 8, 9, 10))
        // 或者
        argsList := []int{2, 3, 4, 5, 6, 7, 8, 9, 10}
        fmt.Println(Add(1,argsList...))
    }

## 任意类型的不定参数

    package main
    
    import (
        "fmt"
    )
    
    /*
    任意类型的不定参数，用interface{}表示
    */
    func MyPrintf(args ...interface{}) {
        for _, arg := range args { //迭代不定参数
            switch arg.(type) {
            case int:
                fmt.Println(arg, "is int")
            case string:
                fmt.Println(arg, "is string")
            case float64:
                fmt.Println(arg, "is float64")
            case bool:
                fmt.Println(arg, " is bool")
            default:
                fmt.Println("Unknow")
            }
        }
    }
    
    func main() {
        MyPrintf(1, "test", 1.5, false, -1.5)
    }

输出结果：

    1 is int
    test is string
    1.5 is float64
    false  is bool
    -1.5 is float64

