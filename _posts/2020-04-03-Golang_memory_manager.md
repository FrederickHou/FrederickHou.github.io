---
layout:     post
title:      Golang 内存分配
subtitle:   
date:       2020-04-08
author:     Frederick
header-img: img/go_memory.jpg
catalog: true
tags:
    - Golang
---

# Golang 内存分配

## 基础概念

Go语言内置运行时（就是runtime），不同于传统的内存分配方式，go为自主管理，最开始是基于 **tcmalloc（thread-caching-malloc）** 架构，后面逐步迭新。自主管理可实现更好的内存使用模式，如内存池、预分配等，从而避免了系统调用所带来的性能问题。

## 基本策略

- 申请一块较大的地址空间（虚拟内存），用于内存分配及管理（golang：spans+bitmap+arena->512M+16G+512G）
- 当空间不足时，向系统申请一块较大的内存，如100KB或者1MB
申请到的内存块按特定的size，被分割成多种小块内存（golang：_NumSizeClasses = 67），并用链表管理起来
- 创建对象时，按照对象大小，从空闲链表中查找到最适合的内存块
- 销毁对象时，将对应的内存块返还空闲链表中以复用
- 空闲内存达到阈值时，返还操作系统

## 基本概念

### 内存块

- **span：** 即上面所说的操作系统分配的大块内存，由多个地址连续的页组成；

    内存分配器按照页数来区分不同大小的span，如以页数为单位将span存放到管理数组中，且以页数作为索引；

    　　span大小并非不变，在没有获取到合适大小的闲置span时，返回页数更多的span，然后进行剪裁，多余的页数构成新的span，放回管理数组；

    　　分配器还可以将相邻的空闲span合并，以构建更大的内存块，减少碎片提供更灵活的分配策略。

    ![span.png](https://upload-images.jianshu.io/upload_images/17904159-5ef8b3da2b764920.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    如上图所示span1占用2个页，span2占用4个页，span3占用3个页。

    **数据结构如下：**

        //mheap.go
        type mspan struct {
            next *mspan   //双向链表 next span in list, or nil if none
            prev *mspan   //previous span in list, or nil if none
            list *mSpanList  //用于调试。TODO: Remove.

            //起始序号 = （address >> _PageShift）
            startAddr uintptr  //address of first byte of span aka s.base()
            npages    uintptr  //number of pages in span

            //待分配的object链表
            manualFreeList gclinkptr  //list of free objects in mSpanManual spans
        }

- **object：** 由span按特定大小切分的小块内存，每一个可存储一个对象；object，按8字节倍数分为n种。如，大小为24的object可存储范围在17~24字节的对象。在造成一些内存浪费的同时减少了小块内存的规格，优化了分配和复用的管理策略，分配器还会将多个微小对象组合到一个object块内，以节约内存。

    span 列表(mspan)分为67个大小级别的多种小块内存(go：_NumSizeClasses = 67)并用链表管理。从8 bytes到32k bytes，这样就可以存储不同大小的对象object。如下图所示：

    ![span-size-classes.png](https://upload-images.jianshu.io/upload_images/17904159-a85a61f69363401e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

按照用途，span面向内部管理，object面向对象分配。

### 存储对象分类

- 小对象（tiny）: size < 16byte;
- 普通对象： 16byte ~ 32K;
- 大对象（large）：size > 32K;

### 内存分配器

**cache：** 每个运行期工作线程都会绑定一个cache，用于无锁object分配(Central组件其实也是一个缓存，但它缓存的不是小对象内存块，而是一组一组的内存page(一个page占4k大小))。每个mcache有大小为67的mspan数组，存储不同级别大小的mspan
**数据结构如下所示：**

    //mcache.go
    type mcache struct{
         以spanClass为索引管理多个用于分配的span
         alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass  
     }
**central：** 全局内存，为所有cache提供切分好的后备span资源。

**数据结构如下所示：**

    //mcentral.go
    type mcentral struct{
        spanclass   spanClass             //规格
        //链表：尚有空闲object的span
        nonempty mSpanList // list of spans with a free object, ie a nonempty free list      
        // 链表：没有空闲object，或已被cache取走的span
        empty    mSpanList // list of spans with no free objects (or cached in an mcache)
    }
**heap：** 全局内存，管理闲置span，需要时间向操作系统申请新内存(堆分配器，以8192byte页进行管理)。

**数据结构如下所示：**

    type mheap struct{
    largealloc  uint64                  // bytes allocated for large objects 
    //页数大于127（>=127）的闲置span链表                                                                                                                     
    largefree   uint64                  // bytes freed for large objects (>maxsmallsize)    
    nlargefree  uint64                  // number of frees for large objects (>maxsmallsize) 
    //页数在127以内的闲置span链表数组                                                                                                                     
    nsmallfree  [_NumSizeClasses]uint64 // number of frees for small objects (<=maxsmallsize)
    //每个central对应一种sizeclass
    central [numSpanClasses]struct {
        mcentral mcentral
        pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
}

## 示例引入

    package main

    type smallStruct struct {
    a, b int64
    c, d float64
    }

    func main() {
    smallAllocation()
    }

    //go:noinline
    func smallAllocation() *smallStruct {
    return &smallStruct{}
    }


**注释：** **“//go:noinline”** 表示该函数禁止进行内联。内联优化是避免栈和抢占检查这些成本的经典优化方法。在没有内联优化的时候new函数会调用newobject在堆上分配内存。


我们使用逃逸分析命令: **"go tool compile "-m" test.go"** 来分析Go的内存分配。

    #go tool compile "-m" test.go
    /test.go:8:6: can inline main
    ./test.go:14:11: &smallStruct literal escapes to heap

使命命令:"go tool compile -S test.go"查看内存分配过程细节。

    #go tool compile -S test.go
       
    0x0000 00000 (./test.go:13)     TEXT    "".smallAllocation(SB), ABIInternal, $24-8
    0x0000 00000 (./test.go:13)     MOVQ    (TLS), CX
    0x0009 00009 (./test.go:13)     CMPQ    SP, 16(CX)
    0x000d 00013 (./test.go:13)     JLS     65
    0x000f 00015 (./test.go:13)     SUBQ    $24, SP
    0x0013 00019 (./test.go:13)     MOVQ    BP, 16(SP)
    0x0018 00024 (./test.go:13)     LEAQ    16(SP), BP
    0x001d 00029 (./test.go:13)     FUNCDATA        $0, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
    0x001d 00029 (./test.go:13)     FUNCDATA        $1, gclocals·69c1753bd5f81501d95132d08af04464(SB)
    0x001d 00029 (./test.go:13)     FUNCDATA        $3, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
    0x001d 00029 (./test.go:14)     PCDATA  $2, $1
    0x001d 00029 (./test.go:14)     PCDATA  $0, $0
    0x001d 00029 (./test.go:14)     LEAQ    type."".smallStruct(SB), AX
    0x0024 00036 (./test.go:14)     PCDATA  $2, $0
    0x0024 00036 (./test.go:14)     MOVQ    AX, (SP)
    0x0028 00040 (./test.go:14)     CALL    runtime.newobject(SB)
    0x002d 00045 (./test.go:14)     PCDATA  $2, $1
    0x002d 00045 (./test.go:14)     MOVQ    8(SP), AX
    0x0032 00050 (./test.go:14)     PCDATA  $2, $0
    0x0032 00050 (./test.go:14)     PCDATA  $0, $1
    0x0032 00050 (./test.go:14)     MOVQ    AX, "".~r0+32(SP)
    0x0037 00055 (./test.go:14)     MOVQ    16(SP), BP
    0x003c 00060 (./test.go:14)     ADDQ    $24, SP
    0x0040 00064 (./test.go:14)     RET
    0x0041 00065 (./test.go:14)     NOP
    0x0041 00065 (./test.go:13)     PCDATA  $0, $-1
    0x0041 00065 (./test.go:13)     PCDATA  $2, $-1
    0x0041 00065 (./test.go:13)     CALL    runtime.morestack_noctxt(SB)

可以看到在运行到smallAllocation函数时调用了runtime.newobject函数来申请的内存。**在此创建过程发生了什么？具体看下面讲解。**

### 微小对象分配

小于32kb的微小对象内存分配。Go 将从叫做mcache的本地缓存中获取内存。这个缓存管理一个span链表(32kb的内存块）称为mspan。其中包含可分配的内存。如下图所示：
![allocation-with-mcache.png](https://upload-images.jianshu.io/upload_images/17904159-53534f89088e5340.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


根据PMG调度模型大家应该知道每个线程M将分配一个处理器P,并且一次最多处理一个goroutine。在分配内存时，当前的goroutine将使用其当前P的本地缓存mcache，在mspan列表中找出第一个空闲的对象。使用本地缓存mcache不需要锁，这样可以使内存分配效率更高。



例如上述示例中结构体占用内存是32bytes 将给分配32bytes span 。则申请示意图如下所示：

![code-demo.png](https://upload-images.jianshu.io/upload_images/17904159-2774d10c0b23d6b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**现在，我们想象一下如果span 再分配的时候没有空闲的内存，将发生什么？**

**1.** 先来看一下这个概念，Go 中central维护同样大小classes span的链表，叫做mcentral，此数据结构用来表示对应大小classes的span中已经被使用的object和空闲的object 链表。如下图所示：

![central-lists-of-spans.png](https://upload-images.jianshu.io/upload_images/17904159-6832962f272102ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据上图可以看出一个mcentral维护两个链表nonempty 和empty 链表。并且每个链表包含当前的span 和下一个span。 nonempty 链表的 span意味着至少有一个可分配的内存。也包含着一些在使用中的内存分配。当垃圾回收器对这块使用中的内存进行回收时，他将清除span这部分内存，在span中此部分将被标记成没有被使用状态。并且将添加回nonempty 链表中去。


**2.** 再回到问题，我们知道这个mcentral后,便可得知如果一个span没有了可分配的内存了，便可以从central链表(全局内存)中重新获取一个。如下图所示：

![span-from-central.png](https://upload-images.jianshu.io/upload_images/17904159-dc3510dd229be832.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

假如P的一个运行的goroutine，系统分配一个8 bytes的内存，通过本地缓存mcache去mspan 链表中申请，如果没有可分配的8 bytes的对象了，那么就会在后备span资源mcentral中去重新申请一个对应size classes的span。

**3.** 那么问题来了，如果central 链表中没有可分配的新span。那么在Go中新的span将会在heap中申请得到，并且将新的span的对象存储状态关联到central链表中去，如下图所示。

![span-from-heap.png](https://upload-images.jianshu.io/upload_images/17904159-e3727b2d0e33793d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当需要时，heap将从系统中申请一片内存(小内存)。此外如果需要更大的内存，heap将从系统中申请一大段内存，叫做arena。(在64位架构系统中将申请64MB,其他的大多数架构系统将申请4MB)。arena也映射spans的内存页。如下图所示

![arena-to-span.png](https://upload-images.jianshu.io/upload_images/17904159-ccd830361bcb768a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 大对象分配

在Go中不用本地缓存 mcache管理大对象内存分配。这些分配通常指大于32kb内存。页内存将直接在heap上申请。

![large-allocation.png](https://upload-images.jianshu.io/upload_images/17904159-05149ac769cb999c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 内存分配示意图

通过以上概念引入，再到举例切入，认识和了解Go的内存分配流程，再从宏观来看一下内存分配，如下图所示。

![memory-allocation.png](https://upload-images.jianshu.io/upload_images/17904159-f923a2783f0235ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 源码分析

根据示例引入的例子，内存分析此看到是runtime.newobject()函数调用，是Go的内存分配内置函数。源码如下：

    //malloc.go
    // implementation of new builtin
    // compiler (both frontend and SSA backend) knows the signature
    // of this function
    func newobject(typ *_type) unsafe.Pointer {
        return mallocgc(typ.size, typ, true)
    }

    // Allocate an object of size bytes.
    // Small objects are allocated from the per-P cache's free lists.
    // Large objects (> 32 kB) are allocated straight from the heap.
    ///分配一个大小为字节的对象。小对象是从per-P缓存的空闲列表中分配的。 大对象（> 32 kB）直接从堆中分配。
    func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
        if gcphase == _GCmarktermination { //垃圾回收有关
            throw("mallocgc called with gcphase == _GCmarktermination")
        }

        if size == 0 {
            return unsafe.Pointer(&zerobase)
        }
        if debug.sbrk != 0 {
            align := uintptr(16)
            if typ != nil {
                align = uintptr(typ.align)
            }
            return persistentalloc(size, align, &memstats.other_sys)  //围绕sysAlloc的包装程序，可以分配小块。没有相关的自由操作。用于功能/类型/调试相关的持久数据。如果align为0，则使用默认的align（当前为8）。返回的内存将被清零。考虑将持久分配的类型标记为go：notinheap。
        }

        // assistG是要为此分配收费的G，如果GC当前未激活，则为n。
        var assistG *g

        ...

        // Set mp.mallocing to keep from being preempted by GC.
        //加锁放防止GC被抢占。
        mp := acquirem()
        if mp.mallocing != 0 {
            throw("malloc deadlock")
        }
        if mp.gsignal == getg() {
            throw("malloc during signal")
        }
        mp.mallocing = 1

        shouldhelpgc := false
        dataSize := size

        //当前线程所绑定的cache
        c := gomcache()
        var x unsafe.Pointer
        // 判断分配的对象是否 是nil或非指针类型
        noscan := typ == nil || typ.kind&kindNoPointers != 0
        //微小对象
        if size <= maxSmallSize {
            //无须扫描非指针微小对象（16）
            if noscan && size < maxTinySize {
                // Tiny allocator.
                //微小的分配器将几个微小的分配请求组合到一个内存块中。当所有子对象均不可访问时，将释放结果存储块。子对象必须是noscan（没有指针），以确保限制可能浪费的内存量。
                //用于合并的存储块的大小（maxTinySize）是可调的。当前设置为16字节.
                //小分配器的主要目标是小字符串和独立的转义变量。在json基准上，分配器将分配数量减少了约12％，并将堆大小减少了约20％。
                off := c.tinyoffset
                // 对齐所需（保守）对齐的小指针。调整偏移量。
                if size&7 == 0 {
                    off = round(off, 8)
                } else if size&3 == 0 {
                    off = round(off, 4)
                } else if size&1 == 0 {
                    off = round(off, 2)
                }
                //如果剩余空间足够.  当前mcache上绑定的tiny 块内存空间足够，直接分配，并返回
                if off+size <= maxTinySize && c.tiny != 0 {
                    // 返回指针，调整偏移量为下次分配做好准备。
                    x = unsafe.Pointer(c.tiny + off)
                    c.tinyoffset = off + size
                    c.local_tinyallocs++
                    mp.mallocing = 0
                    releasem(mp)
                    return x
                }
                //当前mcache上的 tiny 块内存空间不足，分配新的maxTinySize块。就是从sizeclass=2的span.freelist获取一个16字节object。
                span := c.alloc[tinySpanClass]
                // 尝试从 allocCache 获取内存，获取不到返回0
                v := nextFreeFast(span)
                if v == 0 {
                // 没有从 allocCache 获取到内存，netxtFree函数 尝试从 mcentral获取一个新的对应规格的内存块（新span），替换原先内存空间不足的内存块，并分配内存，后面解析 nextFree 函数
                    v, _, shouldhelpgc = c.nextFree(tinySpanClass)
                }
                x = unsafe.Pointer(v)
                (*[2]uint64)(x)[0] = 0
                (*[2]uint64)(x)[1] = 0
                // 对比新旧两个tiny块剩余空间，看看我们是否需要用剩余的自由空间来替换现有的微型块。新块分配后其tinyyoffset = size,因此比对偏移量即可
                if size < c.tinyoffset || c.tiny == 0 {
                    //用新块替换
                    c.tiny = uintptr(x)
                    c.tinyoffset = size
                }
                //消费一个新的完整tiny块
                size = maxTinySize
            } else {
                // 这里开始 正常对象的 内存分配
                
                // 首先查表，以确定 sizeclass
                var sizeclass uint8
                if size <= smallSizeMax-8 {
                    sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
                } else {
                    sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
                }
                size = uintptr(class_to_size[sizeclass])
                spc := makeSpanClass(sizeclass, noscan)
                //找到对应规格的span.freelist，从中提取object
                span := c.alloc[spc]
                // 同小对象分配一样，尝试从 allocCache 获取内存，获取不到返回0
                v := nextFreeFast(span)
                
                //没有可用的object。从central获取新的span。
                if v == 0 {
                    v, span, shouldhelpgc = c.nextFree(spc)
                }
                x = unsafe.Pointer(v)
                if needzero && span.needzero != 0 {
                    memclrNoHeapPointers(unsafe.Pointer(v), size)
                }
            }
        } else {
            // 这里开始大对象的分配

            // 大对象的分配与 小对象 和普通对象 的分配有点不一样，大对象直接从 mheap 上分配
            var s *mspan
            shouldhelpgc = true
            systemstack(func() {
                s = largeAlloc(size, needzero, noscan)
            })
            s.freeindex = 1
            s.allocCount = 1
            //span.start实际由address >> pageshift生成。
            x = unsafe.Pointer(s.base())
            size = s.elemsize
        }

        // bitmap标记...
        // 检查出发条件，启动垃圾回收 ...

        return x
    }


**代码基本思路：**

![code-flow.png](https://upload-images.jianshu.io/upload_images/17904159-4f75203b9a36bd92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

　　1. 判定对象是大对象、小对象还是微小对象。

　　2. 如果是微小对象：
- 直接从 mcache 的alloc 找到对应 classsize 的 mspan；
- 如果当前mspan有足够空间，分配并修改mspan的相关属性（nextFreeFast函数中实现）；
- 如果当前mspan的空间不足，则从 mcentral重新获取一块 对应 classsize的 mspan，替换原先的mspan，然后分配并修改mspan的相关属性；
- 对于微小对象，它不能是指针，因为多个微小对象被组合到一个object里，显然无法应对辣鸡扫描。其次它从span.freelist获取一个16字节的object，然后利用偏移量来记录下一次分配的位置。

　　3. 如果是小对象，内存分配逻辑大致同微小对象：
- 首先查表，以确定 需要分配内存的对象的 sizeclass，并找到 对应 classsize的 mspan；

- 若当前mspan有足够的空间，分配并修改mspan的相关属性（nextFreeFast函数中实现）；
- 若当前mspan没有足够的空间，从 mcentral重新获取一块对应 classsize的 mspan，替换原先的mspan，然后分配并修改mspan的相关属性；

　　4. 如果是大对象，直接从mheap进行分配，这里的实现依靠 largeAlloc 函数实现。

