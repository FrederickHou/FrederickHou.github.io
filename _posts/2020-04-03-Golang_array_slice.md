---
layout:     post
title:      Golang 数组和切片
subtitle:   
date:       2020-04-03
author:     Frederick
header-img: img/bk_go.jpg
catalog: true
tags:
    - Golang
---

# Golang 数组和切片

## 数组

### 介绍

在Go语言中，数组是一种具有**相同类型固定大小**的一种数据结构。

我们先来看看数组的使用，数组类型声明时的方式是 []T ,前面的[]指定数组的大小，T指定数组的类型，如下我们声明了一下数组，数组的大小是3，在没指定数组初始值时数组默认初始值是{0，0，0}

    package main
    import (
        "fmt"
    )
    func main() {
        array1 := [3]int{}
        //我们可以通过如下方式给数组赋值
        array1[0] = 1
        array1[1] = 2
        array1[2] = 3
        fmt.Println(array1)
        //下面这种也是数组声明的一种方式，并且初始化数组的值为{1,2}
        array2 := [2]int {1,2}
        fmt.Println(array2)
    }
    
**结果输出：**

    [1 2 3]
    [1 2]

### 思考

#### 数组可以进行比较吗？

答案：可以，也不可以(分情况)通过以下两个例子进行

**1.示例：**

    package main
    import (
        "fmt"
    )
    func main() {
        array1 := [3]int{1,2,3}
        //我们可以通过如下方式给数组赋值
        fmt.Println(array1)
        //下面这种也是数组声明的一种方式，并且初始化数组的值为{1,2}
        array2 := [3]int {1,2,4}
        fmt.Println(array2)

        if array1 == array2{
            fmt.Println("true")
        }else{
            fmt.Println("false")
        }
    }

**结果输出：**

    false

**注释:** 没有报错，即可以进行比较

**2.示例：**

    package main
    import (
        "fmt"
    )
    func main() {
        array1 := [3]int{1,2,3}
        //我们可以通过如下方式给数组赋值
        fmt.Println(array1)
        //下面这种也是数组声明的一种方式，并且初始化数组的值为{1,2}
        array2 := [2]int {1,2}
        fmt.Println(array2)

        if array1 == array2{
            fmt.Println("true")
        }else{
            fmt.Println("false")
        }
    }

 **结果输出：**

    # command-line-arguments
    .\test.go:13:12: invalid operation: array1 == array2 (mismatched types [3]int and [2]int)

**注释:** 程序报错，即不可以进行比较。

#### 总结：

通过以上两个例子，我们可以得到，首先在进行比较时此类型必须是可比较类型，如 map，[]切片是不可比较类型。在可比较类型的前提下，必须是固定长度且类型相同才可以进行比较。

## 切片

### 介绍

在Go语言中，切片是数组的一种高级运用，相对于数组，切片是一种更加方便，灵活，高效的数据结构。，切片并不存储任何元素而只是对现有数组的引用（不是值拷贝，是指针）

### 定义切片

    package main
    import (
        "fmt"
    )
    func main() {

        array1 := [3]int{1,2,3}
        //将数组下标从1处到下标2处的元素转换为一个切片(前闭后开)
        slice1 := array1[1:2]
        fmt.Println(slice1)

        slice2 := []int{1,2,3}
        fmt.Println(slice2)

        slice3   := make([]int,3,6)
        //给切片赋值
        slice3[0] = 0
        slice3[1] = 1
        slice3[2] = 2
        //通过len([]Type) cap([]Type)两个函数查看切片的长度和容量
        fmt.Println(slice3)
        fmt.Println(len(slice3),cap(slice3))
    }

**输出结果：**

    [2]
    [1 2 3]
    [0 1 2]
    3 6

**注释:** 以上示例用三种方式定义一个切片，（这里大家有个疑问，我不管怎么看都觉得它是一个数组啊）**记住，再go语言中，区别一个变量是数组还是切片，就看有没有定义长度，有定义长度就是数组反之就是切片。** 

## 数组和切片的底层存储

![array_slice.png](https://upload-images.jianshu.io/upload_images/17904159-d4e5fcdbb49d46ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

基于上图我们会发现，切片的底层就是数组，切片是通过指针的形式指向不同数组的位置从而形成不同的切片，切片对本身元素的修改，也会影响到数组和其它的切片。看下面示例代码

    package main
    import (
        "fmt"
    )
    func main() {

        array11 := [5]int{1,2,3,4,5} //[1,2,3,4,5]
        slice11 := array11[0:3] //[1,2,3]
        slice11[0] = 10 //[10,2,3]
        sliceTemp := append(array11[:2],array11[3:]...) //[10,2,4,5]
        slice11[0] = 11 //[11,2,4]
        fmt.Println(array11,slice11,sliceTemp) //[11,2,4,5,5] [11,2,4] [11,2,4,5]

    }
**输出结果：**

    [11 2 4 5 5] [11 2 4] [11 2 4 5]

**注释:** 
第一行：创建了一个长度为5，且存储的元素类型是int的数组array11并且初始化为[1,2,3,4,5] 

第二行：基于数组array11创建了一个切片slice11，切片内容是array11数组的第一个元素到第三个元素的切分。值为[1,2,3]

第三行：将切片的slice11第1个元素重新复制为10，即slice11值为[10,2,3].由于切片是指向数据的指针，因为重新赋值就是将数组对应下标的值改掉。此时数组array11的值为[10,2,3,4,5]

第四行：基于数组array11创建切片sliceTemp，所取的值是数组array11前两个元素[10,2]和后两个值[4,5]所以sliceTemp值为[10,2,4,5]。此时sliceTemp是由于数组array11作为原数组，将其前两个元素保留[10,2]再将后两个元素[4,5]做重新赋值给array11第三个和第四个元素，此时array11的值就发生了变化为[10,2,4,5,5]

第五行：slice11又重新给第1个元素复制为11，由于第四行操作已经改变了array11的值,因此slice11的值也会发生变化为[11,2,4,5,5]，此时slice11的值为[11,2,4]。
sliceTemp的值也变为[11,2,4,5]