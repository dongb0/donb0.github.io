---
layout: post
title: "[os] 进程、线程和协程"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
catalog: true
tags:
  - os
---

> 「现代操作系统-原理与实现」中关于进程、线程、协程的介绍解决了我很多以前看「操作系统概念」产生的疑惑，整理出这份笔记以备查阅，当然还有欠缺，以后慢慢完善

### 进程

**进程**是运行中的程序。不同进程之间的有独立地址空间、数据段/代码段、堆、栈等。进程切换时，需要将PID、进程状态、虚拟内存状态、打开文件、寄存器状态等等信息保存在 PCB （进程控制块）中，然后由调度器选择下一个要执行的进程切换到它的上下文（恢复寄存器中存储的值）开始执行；要继续执行被切换的进程时，就要进入内核态读取PCB中的信息来将进程的状态恢复，然后回到用户态继续执行。

---------

> 曾看到这样的说法，说进程是分配资源的最小单位。觉得有些不对劲，仔细想了想，觉得疑问出在”资源“的范围究竟是什么。如果说是内存、寄存器、文件描述符等，那么似乎确实是（等下，线程之间对寄存器的使用好像是独立的？）。但是CPU是否也应该算是一种资源呢？线程才是CPU调度的最小单位，既然每个线程获得的CPU资源是独立的，那么我觉得就不应该说进程是资源分配的最小单位，或者需要进行补充说明，比如指明”资源“的定义。这里主要问题还是出在对于内核的理解太肤浅，对于进程和线程创建是需要分配什么、操作系统需要做什么工作理解不到位，希望之后通过一些公开课的实验能加深这方面的了解吧。

---------

### 线程

因为进程创建时，需要创建独立的虚拟地址空间，初始化堆栈等，开销较大；另外进程的地址空间是独立的，进程之间通信也比较麻烦（通过共享内存或管道等方式？//TODO），因此操作系统设计人员在进程内部引入了**线程**。线程共享进程内部的地址空间，但是拥有自己的栈和自己执行的上下文状态。引入线程之后，线程就取代进程成为操作系统调度的最小单元。

切换时也需要保存线程执行的上下文到TCB中，但通常认为线程间切换的开销比进程开销小。以我目前的了解来看，虽然进程和线程切换都需要进入内核态读取PCB/TCB，但是进程切换意味着页表映射的改变，//* *TODO*:虚拟内存需要更多了解 *//，一二三级缓存和TLB中会产生较多 miss，这贡献了进程切换中相当大一部分时间消耗，如果内存空间不足，置换页会产生更大的开销；而同一个进程中的线程发生上下文切换时，因为共享内存空间，不太容易产生缓存miss的情况。

线程的栈在x86-64架构下默认为2MB大小，函数内的局部变量都在栈上分配，如果创建过大的局部变量或者递归层数太深就会出现经典的 stackoverflow 错误（本以为golang的栈会动态扩容，搜索了一下发现还是能造出stackoverflow，参见[这里](http://vearne.cc/archives/1464)）

憨憨如我以前还在想为什么线程之间不需要通信，后来发现线程共享地址空间，只要解决竞争访问全局数据的问题就行（一般通过互斥锁、读写锁、条件变量等），所以一般我们说**进程**之间通信，**线程**之间同步。当然这里说的是同属一个进程的线程之间，如果与其他进程下的线程进行交互，则变为进程之间通信问题。

在我的电脑上做了点测试（参考自这个[博客][1]），首先`arch`输出`x86_64`，说明是x86_64架构的；`ulimit -a`查看到默认栈大小为`8192 kb`，就是8MB。然后我写了个死循环的程序，在运行的时候去看 `/proc/[pid]/maps` 文件，却得到的是这样的结果

![screenshot-1](/img/in-post/post-thread-insight/proc-maps.png)

这里看到的 stack size 是136KB，跟前面说的 x86_64 架构默认2MB以及刚刚查的 Linux 默认栈大小 8MB 都对不上。后来发现 maps 存的是虚拟内存的映射情况，应该不是我想要的能直接看到实际 stack 大小。想想干脆直接在 main 函数里开辟一个8MB的数组看结果，结果确实直接 stackoverflow 了；如果开7.5MB差不多能开得下，侧面证明 Linux 的stack大小是8MB，虽然 x86 默认是 2MB 但是 Linux 应该是创建线程是传入了 ulimit 设定的参数改为了8MB。

// TODO： /proc/\[pid]/maps 其他字段含义和用途

### 协程

Linux和Windows的线程都是内核态与用户态一对一映射的，操作系统会对内核线程的数量做限制。有些应用场景需要大量 short-lived 线程来响应请求，比如Web应用，这些线程实际使用的内存并不多，使用默认大小的栈空间导致创建线程的总数也受到限制（加上操作系统对线程数量的限制），导致应用的并发数受限。

另外一对一模型其实相当于线程的调度都由操作系统控制，应用程序比操作系统更了解程序执行的语义信息，也许会比操作系统做出更合理的调度（想起数据库自己实现BufferPool也基于类似的考虑），所以协程的使用逐渐多了起来。

线程切换时要进入内核态执行代码（我理解为跳转到线程中在用户态时不可见的一块区域，也就是某块内核区域的代码执行，这当然是有开销的）；但是协程切换只用保存用户态的TCB（如golang中的g结构体），不涉及内核态和用户态的切换，因此比内核线程切换开销要小（协程切换时内核线程一直在干活，线程切换时两个线程换班）。

### 扩展：Goroutine的调度

前面提到的协程有些可能是操作系统本身支持的（比如POSIX的ucontext，//* 也许要更新更多说明 *//），而一些编程语言也实现了协程，如golang在编程语言层面提供了对协程的支持，它实现了自己的调度器，通过GMP模型组织协程的调度[^1]。

G -> goroutine，一个存储用户态线程状态的结构体，也可理解为一个任务  
M -> machine，代表操作系统线程  
P -> processor，实际上是一个线程本地的 goroutine 的队列，每个 P 会与一个 M 绑定，负责调度队列中的 goroutine

特殊结构：  
g0 -> 每个 M 用来调度的 goroutine，我的理解是当正在执行的 goroutine 结束运行或阻塞时，就轮到 g0 运行来选择下一个要执行的 goroutine  
m0 -> 初始的 M 线程，创建第一个 g 后作为普通 m 使用

概括goroutine调度的过程：一般来说会根据CPU核心数量创建等数量的P，然后每个P与一个M绑定，M从P的任务队列中取出 g 开始执行。正在执行的G如果创建了新的 g，就存到P的本地队列中，每个P最多存放256个 g，超出该数量的 g 会放到全局队列中。当一个P没有任务可以执行时，会先到全局队列中寻找等待执行的 g；如果全局队列也为空，则尝试从其他P中偷取队列前一半的 g 放到自己队列中执行。

如果某个M执行 g 时进入了阻塞，则P会寻找一个新的空闲线程绑定，开始执行剩下的 g。当这个 M 从阻塞中恢复时，它也会尝试绑定一个空闲的P继续执行（猜想此时造成阻塞的 g 应该放到队列尾，比较符合队列的特性，因为实际底层数据结构是一个数组实现的循环队列；如果有什么证据表明放到队列头更合适的话那也可以考虑），如果没有的话就把 g 放入全局队列，M陷入休眠。

golang协程的栈初始大小为2KB，会在执行过程中动态扩容或缩小，不过还是有上限，为1GB。  
// TODO：该上限应该也可以调吧？待验证

The End

---------------------

说起来，我只想简简单单看一看协程切换时，都需要保存什么状态，比如会保存什么寄存器状态、goroutine状态之类的信息，为什么没有一篇博客能直截了当告诉我呢？快看吐了我要。

那么根据我的理解，协程切换时，所有的上下文信息都保存 g 结构体里，粗略看了一眼源码几乎全都是各种指针、整型和 bool，还有栈指针以及偏移量等，如果用户态线程切换要拷贝这个看起来不大的结构体的话，那么这部分开销确实没有多少。

有点明白了，进程切换除了要切换寄存器等上下文信息，通常还要伴随着页表切换？// TLB cache miss   // TODO：要更详细
而线程切换比起协程切换，主要的开销在于线程切换要进入内核态，内核态和用户态之间切换的开销

------------

// TODO: 用户态和内核态的定义

进程创建需要消耗什么： 开辟自己的虚拟内存空间（需要分配多少内存吗？）  
进程间切换的开销有什么：

线程创建需要消耗什么： 共享进程的内存空间，但是需要开辟独立的栈  
线程间切换的开销有什么：


-----------

### 参考

[^1]: [GMP原理与调度分析](https://www.jianshu.com/p/fa696563c38a?utm_campaign=studygolang.com&utm_medium=studygolang.com&utm_source=studygolang.com)

[1]: https://blog.csdn.net/liyuanyes/article/details/44097731