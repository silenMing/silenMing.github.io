---
layout:     post
title:      多线程模型之 - golang G的结构及创建过程
subtitle:   
date:       2018-11-25
author:     BY
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - 多线程
    - Go
---

> 上一章我们分析了计算机操作系统中的线程模型，今天我们从源码的角度来具体分析下golang中的线程模型的实现。

> 上一章可见这边文章[多线程模型之 - 操作系统线程模型](https://silenming.github.io/2018/11/10/%E5%A4%9A%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B%E4%B9%8B-%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B/)

#### Go的线程实现模型

在操作系统提供的内核线程的基础上，Go构建了特有的两级线程模型，我先借助于《Go并发编程实战》中 郝林大神的两张图


![](http://silenblog.oss-cn-beijing.aliyuncs.com/005.png)

![](http://silenblog.oss-cn-beijing.aliyuncs.com/006.png)

- M:工作线程
- P:执行GO代码片所需的资源（上下文环境）
- G:就是我们的goroutine代表的代码块


#### goruntine创建源码剖析

我们从创建一个goroutine开始执行来剖析，打开源码包runtime,在包下有一系列的asm.s的汇编文件，这些文件是针对不同CPU厂商的系统架构做的一系列适配，比如intel,AMD的X86架构,比如Mips架构等等，golang都对他们做了适配。

我们就从AMD 64位的适配开始看吧，下面的是启动过程的汇编代码


![](http://silenblog.oss-cn-beijing.aliyuncs.com/WX20190224-131602%402x.png)

这边主要有这个几个操作 args(),osinit(),schedinit(),mainPC(),newproc()


前面的几个方法，args(),osinit(),schedinit()这些都是初始化的一些操作，比如说加载CPU，初始化内核态线程。。

重点看下newproc

![](http://silenblog.oss-cn-beijing.aliyuncs.com/WX20190224-134203%402x.png)
这里有3步操作

1. 计算额外参数的地址，把他加到参数栈中
2. 获取调用端的地址 
3. 使用systemstack去调用newproc1,systemstack的作用是切换栈空间，在执行newproc1完成后，再切回原来的栈空间

再来分析下newproc1看他其中的流程

```java
_g_ := getg()

```
- 调用getg()获取当前的g,也就是goroutine

```java
_g_.m.locks++ // disable preemption because it can be holding p in a local var
```
- 给这个goroutine所对应的m 进行加锁，目的是为了避免出现抢占这个线程的情况

```java
newg := gfget(_p_)
if newg == nil {
    newg = malg(_StackMin)
    casgstatus(newg, _Gidle, _Gdead)
    allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
}
```

- 如果m没有拥有这个g,那么我们新建一个
- 注意：在这我需要先设置G的状态为终止，要不然在创建过程中，如果发生GC了，那么他可能会被回收

```java

    memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
    newg.sched.sp = sp
    newg.stktopsp = sp
    newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
    newg.sched.g = guintptr(unsafe.Pointer(newg))
    gostartcallfn(&newg.sched, fn)
    
    //gostartcallfn 的调用方法最终
    func gostartcall(buf *gobuf, fn, ctxt unsafe.Pointer) {
    sp := buf.sp
    if sys.RegSize > sys.PtrSize {
        sp -= sys.PtrSize
        *(*uintptr)(unsafe.Pointer(sp)) = 0
    }
    sp -= sys.PtrSize
    *(*uintptr)(unsafe.Pointer(sp)) = buf.pc
    buf.sp = sp
    buf.pc = uintptr(fn)
    buf.ctxt = ctxt
}
```
- 将返回地址复制到g的栈上
- 设置sched的参数，然后将sched传入gostartcall,构建以下runnable所需要的参数

```java
runqput(_p_, newg, true)
```

- runqput加入到g所运行的队列

```java
if next {
    retryNext:
        oldnext := _p_.runnext
        if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
            goto retryNext
        }
        if oldnext == 0 {
            return
        }
        // Kick the old runnext out to the regular run queue.
        gp = oldnext.ptr()
    }
```
看上面这个代码，每次的线程切换，我们通过CAS的方式来切换，如果切换失败，那么他会自旋
直到成功为止

```java
if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
    wakep()
}
_g_.m.locks--
if _g_.m.locks == 0 && _g_.preempt { // restore the preemption request in case we've cleared it in newstack
    _g_.stackguard0 = stackPreempt
}

```

- 如果没有空闲的P ，当前无自旋的M，并且主函数已经执行，则需要唤醒或者新建M，也就是我们需要往内核态新申请一个线程

以上就是G创建的一些主要流程

当然在G创建完成后，还需要创建M，创建调度器，篇幅有限，我还没有完全理解那，下次有时间理解透了再写

参考文献：
《Go并发编程实战》 人民邮电出版社 郝林















