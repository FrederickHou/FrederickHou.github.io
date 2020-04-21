---
layout:     post
title:      Golang range 的坑
subtitle:   
date:       2020-04-14
author:     Frederick
header-img: img/go.jpg
categories:
    - 技术笔记
    - Golang 笔记
tags:
    - Golang
toc: true
cover: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1587104469995&di=78cdb590d86d269986999e34b809a620&imgtype=0&src=http%3A%2F%2Fimg2.imgtn.bdimg.com%2Fit%2Fu%3D2094856847%2C1485015268%26fm%3D214%26gp%3D0.jpg
---

#  Golang range 的坑

使用 Go 语言经常会犯的错误比如当我们在遍历一个数组时，如果获取 range 返回变量的地址并保存到另一个数组或者哈希时，就会遇到令人困惑的现象：

## 错误示例

    package main
    import (
        "fmt"
    )
    func main() {
        arr := []int{1, 2, 3}
        newArr := []*int{}
        for _, v := range arr {
            newArr = append(newArr, &v)
        }
        for _, v := range newArr {
            fmt.Println(*v)
        }
    }

**输出**

    3
    3
    3

**原因**

对于所有的 range 循环，Go 语言都会在编译期将原切片或者数组赋值给一个新的变量 ha，在赋值的过程中就发生了拷贝，所以我们遍历的切片已经不是原始的切片变量了。

而遇到这种同时遍历索引和元素的 range 循环时，Go 语言会额外创建一个新的 v2 变量存储切片中的元素，循环中使用的这个变量 v2 会在每一次迭代被重新赋值而覆盖，在赋值时也发生了拷贝。

## 正确示例

    package main
    import (
        "fmt"
    )
    func main() {
        arr := []int{1, 2, 3}
        newArr := []*int{}
        for i, _ := range arr {
            newArr = append(newArr, &arr[i])
        }
        for _, v := range newArr {
            fmt.Println(*v)
        }
    }

**输出**

    1
    2
    3