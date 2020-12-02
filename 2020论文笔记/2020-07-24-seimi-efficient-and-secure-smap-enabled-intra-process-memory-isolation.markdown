---
layout: post
title: "SEIMI: Efficient and Secure SMAP-Enabled Intra-process Memory Isolation"
date: 2020-07-24 00:50:02 +0800
comments: true
categories: 
---

> 作者：Zhe Wang, Chenggang Wu, Mengyao Xie, Yinqian Zhang Kangjie Lu Xiaofeng Zhang, Yuanming Lai, Yan Kang Min Yang
>
> 单位： State Key Laboratory of Computer Architecture, Institute of Computing Technology, Chinese Academy of Sciences,  University of Chinese Academy of Sciences,  The Ohio State University, University of Minnesota, Fudan University
>
> 会议：IEEE Symposium on Security and Privacy 2020
>
> 原文：https://www-users.cs.umn.edu/~kjlu/papers/seimi.pdf

# Abstract

为了防御内存攻击，研究者近年来提出了各种各样的内存防御技术。比如 CFI （control-flow integrity）控制流完整性， CPI (code-pointer integrity) 指针完整性, Shadow Stack 。运用了这些技术之后，确实能够大大缓解内存攻击带来的危害。但是随着这些防御技术的出现，不少研究者也发现了应对这些防御的攻击技术，比如说针对 CFI 的 data-oriented 攻击等等。为了实现这些攻击，一般情况下都需要泄露一定量的关键数据。于是研究者为了应对这些攻击，往往会应用进程内隔离技术来保护关键数据，防止它们被恶意的攻击者访问甚至是修改。但是这些内存隔离技术往往会带来不少的性能损耗，以致于大多数情况下它们都无法被应用在实际环境之中。作者为了缓解这个问题，结合虚拟化技术以及 SMAP 技术实现了一种非常高效的内存隔离技术。

<!-- more -->

# Background

### 进程内内存隔离

内存隔离技术分为两种，一种叫做 address-based isolation ，另一种叫做 domain-based isolation。作者采用的内存隔离技术属于后者。 domain-based isolation 简单来说就是将所有的关键数据集中放在一个内存区域之中，当需要访问这些数据的时候，需要获取相应的权限。访问结束之后，相应的权限将会被回收。所以实现这一隔离技术带来的开销主要在于权限的赋予以及回收的过程中带来的执行开销。而作者实现的这种隔离技术，因为主要与硬件打交道，所以能够很大程度上减少隔离技术带来的性能开销。

### Intel VT-x Extension

VT-x 是因特尔开发的虚拟化技术，应用于 X86架构。VT-x为处理器增加了两种操作模式：VMX root operation 和 VMX non-root operation，两种操作模式都支持Ring 0 ~ Ring 3 这 4 个特权级。这款技术的初衷是为了加速虚拟机的运行。虚拟机将会运行在 non-root 模式下，而宿主机以及 VMM  将会运行在 root 模式之下。VMM 通过操作 VMCS 结构体来进行 root 模式以及 non-root 模式之间的切换。

### SMAP 保护技术

![image-20200609204736480](/images/2020-07-24/image-20200609204736480.png)

该技术防止运行在 ring-0（kernel) 的代码访问 ring-3(应用层) 的数据。该技术的实现方法实在页表中针对每个页面设置了一个标志位。当标志位为 U (user-mode Page)， 则表示该数据页属于应用层，如果 ring-0 的代码访问它将会出错。当标志位为 S （supervisor-mode Page)  ,该数据页就可以被内核代码访问。



# Overview



### 攻击模型

由于作者提出的这项技术的目的是为了防止内存防御技术被绕过，所以 SEIMI 需要结合内存防御技术一起运行。作者对攻击目标做出如下假设。

1. 目标程序存在一些可以被攻击者利用的内存漏洞。

2. 目标程序被内存防御技术保护（ CFI 等）。

3. 为了绕过这些内存防御技术攻击者必须先绕过 SEIMI，也就是说攻击者无法先绕过这些防御技术来关闭 SEIMI。

   

### 高层设计

作者从 SMAP 技术中获得到了灵感。假如能够将应用层程序运行在0环，那么就能利用 SMAP 技术实现内存隔离。

![image-20200609204736480](/images/2020-07-24/image-20200609204736480.png)

作者的想法是将需要保护的数据的内存页的标志位设置为 U（User-mode Page)，所以在 SMAP 开启的时候无法在内核态中访问该内存页中的数据 ，同时将正常的数据页设置成 S。当一个线程需要访问受保护的数据（U page) 的时候，必须先关掉 SMAP 保护，然后才能访问这些数据。在访问完这些数据之后，再将SMAP保护关闭，防止之后的代码访问这些数据。

![image-20200609210821658](/images/2020-07-24/image-20200609210821658.png)

为了实现这一目标，作者将这种开启来 SEIMI 保护的应用程序运行在 VMX non-root 模式下的0环，这样就可以在应用程序中开启和关闭SMAP。在0环中，开启和关闭 SMAP 十分简便，只需要运行 STAC / CLAC 这两条指令就可以开启关闭 SMAP。

正常的应用程序和 kernel 则运行在 root 模式下的0环以及3环，与正常的使用情况没有差别，所以这项 SEIMI 技术对正常的进程以及内核的执行不会产生任何影响。为了实现 SEIMI ，作者在内核中开发了一个叫做 SEIMI 的内核模块，用来处理 SEIMI 应用程序的执行。

此外，上面提到的是 SEIMI 应用程序运行时的状态。除了运行时的支持，还需要有编译时的支持。

![image-20200609211232296](/images/2020-07-24/image-20200609211232296.png)

于是作者开发了一个库，库里面提供了各种各样的 API，用与设置需要保护的内存。使用了库的应用程序将会被标记为开启了 SEIMI 保护的程序。于是在内核中执行 execve 系统调用的时候，将会将普通的应用程序与开启了 SEIMI 保护的应用程序区分开来。对于 SEIMI 保护的应用程序，将会使用虚拟化技术使其运行在 non-root 模式的 0 环下。

```
sa_alloc()   do_mmap()

sa_free()    do_munmap()
```


### 关键问题

作者提出了三个需要解决的关键疑问。

1. 如何区分读和写的保护。一旦开启了 SMAP ，受到保护的页面是不可读也不可写的。但是有些情况下仅需要完整性的保护，虽然不允许非授权指令修改数据读数据。因为有些数据如果既不能读也不能写的话，大量的读取数据操作都会触发 SMAP 的开启与关闭，将会消耗大量的时间，为了缓解这一问题，需要额外的机制区分读和写的保护。
2. 如何防止关键的内核结构体数据被泄露。页表，段表，这些数据结构一般都是放在 kernel 中。虽然受 SEIMI 保护的程序是运行在内核态，但是它们不应该被允许操作这些数据结构。否则攻击者将会有机会通过修改页表等操作实现攻击。
3. 如何防止程序滥用内核中的特权指令。因为此时 CPU 运行在0环，正常情况下是可以执行内核态中的特权指令的。但是 SEIMI 程序虽然是允许在0环，但是它还是个应用层程序。所以需要区分这些特权指令，并阻止 SEIMI 进程执行这些指令，否则同样攻击者可以很容易的通过修改关键寄存器 （CR0） 等实现攻击。



# 问题解决方案及一些设计细节



### 读写保护的分离

作者通过内存共享的方式解决了这个问题。简单来说，就是让两个虚拟页同时映射到一块受到保护的物理页中。其中一块虚拟页是具有仅读权限的 S（ supervisor-mode Page）标记页，所有的对于该数据的读操作都可以直接通过该虚拟页读取。另一块虚拟页是具有读写权限的 U （user-mode Page）标记页，写操作都通过这个页面进行操作，因此在进行写操作时需要先关闭 SMAP 。



### 如何防止关键的内核结构体数据被泄露

实现的方式也比较简单，就是不将这些数据结构映射到页表中。所以 SEIMI 程序即使是运行在内核态，它们也无法访问这些结构体。只有 VMX root 模式下的SEIMI 管理器才能读写这些结构体。



### 如何防止程序滥用内核中的特权指令

首先的第一步就是要识别出程序的特权指令。作者的实现方法是设计了一个测试程序，在测试程序中逐条运行 SEIMI 程序中的指令，并为这些指令随机设置操作数。一旦 CPU 触发了 invalid code 错误，则认为这是一个特权指令，于是将它标记起来。

识别了特权指令之后，紧接着作者采用了分类讨论的思想，针对它们的不同特性对它们进行分类，然后针对不同的类别对它们采取了不同的操作。文章的不少篇幅也是在分类讨论针对不同类型的指令提出了不同的解决方式。

![image-20200609223104856](/images/2020-07-24/image-20200609223104856.png)

作者将指令的类型分为了20种类型，并将这20种类型划分成了三大类。第一种类型的指令一旦被运行，将会触发 VM-EXIT，控制流将会被转交给 SEIMI 的内核模块控制器。第二种的指令运行时，什么都不会发生。而运行第三种指令，将会触发 CPU 的中断，并终止运行。具体实现的方式可以查阅 paper 中               **IV. SECURELY EXECUTING USER CODE IN RING 0** 这一节的内容。



### 系统调用等中断的执行

此外还有一些实现细节需要被提及。因为 SEIMI 程序虽然运行在内核态，但它没有自己的内核，所以系统调用的时候需要使用宿主机的内核。所以 SEIMI 运行前会将自己的中断向量表指向的指令设置成相应的 VMCALL 指令，一旦调用 syscall 软中断，或者处理其它中断（ 如时钟中断 ）等，将会调用相应的 VMCALL 指令，将控制流交给 seimi 的内核控制器相应的代码，然后再由控制器将这些 syscall 和中断转交给宿主机的 kernel 执行。



# Evaluation

**测试环境 ：Ubuntu 18.04 (Kernel 4.20.3) that runs on a 2.10 GHz Intel(R) Xeon(R) Gold 6130 CPU with 32 cores and 32GB RAM**

![image-20200609231411460](/images/2020-07-24/image-20200609231411460.png)



 ## microbenchmark 测试

因为所有的系统调用都需要通过 seimi kernel module 来转发，所以会带来一定的性能消耗。作者首先想测试的是 seimi 的这些内核操作的引入额外开销。

![image-20200609232054117](/images/2020-07-24/image-20200609232054117.png)

  																			与进程相关的操作的开销

![image-20200609232204846](/images/2020-07-24/image-20200609232204846.png)

​																					上下文切换的开销 

（P 代表运行的进程数，K 代表 woking sets size），总之进程越多开销越大。

![image-20200609232746874](/images/2020-07-24/image-20200609232746874.png)

​																						文件操作的开销

![image-20200609232852600](/images/2020-07-24/image-20200609232852600.png)

​																						通信操作的开销



## macrobenchmark  测试

然后作者将4种隔离技术（IH \ MPX \ MPK \SEMI ) 分别作用于四种防护技术 （ O - CFI \  SS \ CPI \ AG ），对比宏观运行的执行效率。

![image-20200609233019974](/images/2020-07-24/image-20200609233019974.png)

​																								OCFI

![image-20200609233119228](/images/2020-07-24/image-20200609233119228.png)

​																								SS

![image-20200609233054779](/images/2020-07-24/image-20200609233054779.png)

​																								CPI

![image-20200609233134932](/images/2020-07-24/image-20200609233134932.png)

​																								AG

总结来说，与 MPK-based 隔离技术相比，SEIMI平均的开销要少33.97%

与MPX-based 隔离技术相比，SEIMI平均开销少了42.3%



## realworld 测试

作者选用了12款现实软件对性能进行了测试

**web servers**: Nginx-1.4.0, Apache-2.4.38, Lighttpd-1.4 and Openlitespeed-1.4.51. 

**databases**: MySQL5.5.14, SQLite-3.7.5, Redis-3.2.6, and Memcached-1.5.10.

**Javescript engines**: we use ChakraCore (release-1.11), V8 (release-8.0), JavaScriptCore (v.251703), and SpiderMonkey (v.59.0a1.0)

![image-20200609233952719](/images/2020-07-24/image-20200609233952719.png)



测试结果显示，SEIMI 在所有32个测试中，性能均高于MPK，在28个测试中，性能高于MPX。



# 总结

1. 作者用一种创新的方式实现了 domain-based 的进程内隔离技术
2. 该技术的效率比以往的 domain-based 以及 address-based 隔离技术的效率要高不少
3. 因为这种用虚拟化技术将用户进程运行在内核中的思路能够充分利用.硬件进行防护，所以作者认为这种思想为以后的研究带来了一定的借鉴意义