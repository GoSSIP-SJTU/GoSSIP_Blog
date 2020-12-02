---
layout: post
title: "Effective Static Analysis of Concurrency Use-After-Free Bugs in Linux Device Drivers"
date: 2019-11-22 03:51:24 -0500
comments: true
categories: 
---
> 作者：Jia-Ju Bai, Julia Lawall, Qiu-Liang Chen, Shi-Min Hu
>
> 单位：Tsinghua University, Sorbonne University/Inria/LIP6
>
> 出处：USENIX ATC 2019
>
> 资料：[Paper](https://www.usenix.org/system/files/woot19-paper_schink.pdf)

---

## 1. Abstract & Introduction

作者使用了一种静态分析方法来有效检测Linux内核驱动中的并发类型UAF Bug (Use-After-Free)，这个方法作者命名为DCUAF。作者在Linux-4.19上进行了评估，成功发现了640个真实存在的并发UAF，随机选了130个报告给Linux开发者，其中95个已经被确认。

UAF可以分为并发UAF和顺序UAF，这个工作研究的对象是并发UAF。并发UAF相较于顺序UAF，更难被触发，也更难被发现。作者分析了Linux内核的Commit记录，42%的修复UAF的Commit都是跟并发有关，并且这类UAF几乎都是靠人工检查或者运行时测试发现的。

<!--more-->

### 1.1 Linux驱动接口模型

如下所示，结构体里定义了一些函数指针，都是驱动的入口。

![](/images/2019-11-22/Untitled-97503e9d-a098-4508-9dae-b934860ae15d.png)

### 1.2 并发UAF

下图例子中，cw1200_hw_scan和cw1200_bss_info_changed是可以并发执行的。在cw1200_hw_scan里在没有获得锁的情况下就进行释放，因此是可以与cw1200_bss_info_changed里的内存读操作并发执行的，从而造成了Bug。

![](/images/2019-11-22/Untitled-0940765b-8a07-43b4-8cf9-46ee8d32d99a.png)

### 1.3 Kernel Commits

作者通过`git --grep`找出了跟UAF相关的949个Commit，并从中挑出了跟驱动相关的 (driver和sound目录下的）。这里得出的结论是，49%的UAF相关Commit都是跟设备驱动相关，这其中42%的跟并发相关。大约65%的驱动UAF Commit里都提到了是用KASAN、Syzkaller、Coverity、Coccinelle、LDV这些工具找到的。

![](/images/2019-11-22/Untitled-82ebbced-4d48-4e33-bd9b-c657f89c27a5.png)

其中KASAN和Syzkaller是运行时测试工具，Coverity、Coccinelle、LDV是静态分析工具，仅有7个UAF Fix是这些静态分析工具找到的，并且没有与并发相关的UAF。

## 2. 关键技术

作者关于发现并发式UAF的想法：

1. 找到可以并发执行的函数对 (Concurrent Function Pairs)

2. 在这些函数对上进行Lockset Analysis，找出并发UAF。

在实现这个想法的过程中，会遇到的问题：

1. 如何提取并发执行的函数对。
2. 代码分析的准确性和效率，Lockset分析会很花时间。

对于第一个问题，作者采取了Local-global的策略。

对于第二个问题，作者提出使用Summary-based Lockset Analysis。

### 2.1 Local-Global Strategy

观察图2的例子，两个可以并发的函数都通过`mutex_lock`来获得锁，并且锁变量都是`priv→conf_mutex`。通过这个信息，可以判断`hw_scan`和`bss_info_changed`是可以并发的接口的对。但是这种推断方法有可能会问题，例如如下两个常见的例子：

Case1：两个请求相同锁的函数是有可能永远不会被并发执行的，如下图所示。

![](/images/2019-11-22/Untitled-e8dca96f-f5fe-4a66-bf1b-5fc304902486.png)

Case2：对于给定的两个驱动接口，仅有少量的驱动是获取相同的锁的。如下图所示是统计结果。Both代表同时有第一列和第二列的驱动接口的驱动文件数量，Concurrent表示其中请求相同锁的驱动文件数。并且probe是用来初始化一个SPI设备，remove是用来删除一个SPI设备的。显然这两个设备接口是没法并发执行的。

![](/images/2019-11-22/Untitled-7e4792b2-d3d8-414e-8693-cd55a724b95c.png)

为了解决这个问题，作者收集了每个驱动使用锁的信息，作为"Local Information"，然后将所有驱动的这些信息进行结合，从而进行Global的统计分析。

Local Stage: 在这个阶段，提取本地可并发的接口对。

Step1如下图算法所示。遍历驱动里所有使用锁的地方，判断使用的锁是否为别名，如果是，就认为是可并发函数。

![](/images/2019-11-22/Untitled-4613bbd6-765d-40fb-9abb-6a5ffd52dd51.png)

Step2如下图算法所示。判断算法1中所得到的可能为可并发函数对的调用者，判断是否可能是同一个调用路径下的，如果是就删除。

![](/images/2019-11-22/Untitled-54d85cbe-9b90-45e0-b79d-ea50d4f6143e.png)

Step3如下图算法所示获得接口对。

![](/images/2019-11-22/Untitled-bf8f9be8-f938-4d98-b287-bd935765284a.png)

Global Stage: 在这个阶段，已经有了Local Stage收集的信息，通过统计分析来提取出Global的可并发接口对。如下图算法所示提取出比例，只有比例是大于设定的`R`值，才会保留下来。

![](/images/2019-11-22/Untitled-d10c823c-a09a-4853-9c56-0c1b9aec6a52.png)

### 2.2 Summary-Based Lockset Analysis

这一步的分析具备如下特点：

1. 上下文敏感、过程间分析。
2. 流敏感，目的是改善准确性。
3. Function Summaries，减少重复的分析，提高效率。
4. Field-based，专注于存在结构体里的变量。

分两步进行，Step1: 收集每一个变量访问的LockSet，在收集时，使用Function Summaries来处理被调用的函数。每一个Function Summary用函数名、源文件名、存放函数里所有变量访问的Set，以及到达变量访问的路径。

如下图所示算法，

![](/images/2019-11-22/Untitled-41aa61bd-fb8b-4bd4-b78c-6abc4ce3120f.png)

Step2: 对于两个并发接口里的两个变量访问，满足以下条件则认为是Bug：

1. 访问的变量是相同的（别名）。
2. Lockset的交集是空的。
3. 其中一个被访问的变量被传入了类似Free的函数（如果两个都是被Free的，就是Double-Free）。

## 3. 实现

作者将自己的分析方法命名为DCUAF。DCUAF使用Clang6.0，在LLVM IR上分析内核驱动代码。如下图是DCUAF的整体架构。

![](/images/2019-11-22/Untitled-5803c6ce-6246-4b1b-8475-8d17c78d5df1.png)

DCUAF总共分为4个阶段。

1. 源代码编译。将驱动源代码编译生成LLVM Bitcode。这里需要考虑跨模块调用的问题，DCUAF需要记录某些跨模块的函数调用所需要函数定义所在的文件。
2. 代码信息收集。在这个阶段，会收集的信息包括每个函数定义的名字和位置。中端处理函数和驱动函数存放的结构体等。
3. 可并发函数对提取。使用local-global的策略来分析LLVM的bitcode，对每个驱动进行分析，找出可能的可并发函数对。
4. Bug检测。基于前面获得的信息，通过Summary-based Lockset Analyssis分析，并检测出并发式UAF。这一步还需要去重，因为有些找出的Bug可能仅仅是调用路径不同。但是变量访问位置是一样的。

## 4. 评估

在Linux-3.14 (2014.3发布) 和Linux-4.19 (2018.10发布) 上进行了测试。

![](/images/2019-11-22/Untitled-39030a9b-4ee7-4afa-a9f4-2095bc6db167.png)

实验环境：

- Lenovo x86-64
- Intel i5-3470@3.20G处理器
- 8G内存

DCUAF可以并行工作，作者设置了使用4个线程同时工作。

### 4.1 提取可并发函数对

这一步使用Local-global策略提取可并发函数对，在Global策略里设置`R=0.2`。如下所示是处理时的信息。

![](/images/2019-11-22/Untitled-7c99d892-0e07-4d2c-9097-3e0f47942da4.png)

如下图所示是提取出结果的一部分示例。

![](/images/2019-11-22/Untitled-0ee3e5bb-fc2d-4cc5-bb1d-35594884465f.png)

### 4.2 检测Bug

作者用Linux-3.14来评估DCUAF是否能检测出已知Bug，用Linux-4.19来检测DCUAF是否能检测出未知Bug。

在Linux-3.14里，结果显示DCUAF可以找出Linux内核里559个并发的UAF Bug，其中526个经过核对是已知Bug，存在在108个不同源文件里，这其中，有35个在Linux-4.19里已经被修复。因此证明DCUAF是可以检测出已知Bug的。

在Linux-4.19里，结果显示DCUAF找出了679个并发UAF，其中640个被核对确实是Bug，存在于132个不同源文件里。在这些Bug中，372是在Linux-3.14中也找到了，因此是存在了4.5年之久。作者随机选取了这其中的130个Bug报告给Linux开发者，其中95个已经被确认了。因此证明DCUAF是可以发现新Bug的。

另外，在这些找到的Bug里，也有很多是Double Free类型Bug。

超过60%的被验证过的Bug是存在于网络、TTY、字符、ISDN驱动里。作者发现，在这些驱动里，能并发执行的函数也特别多。

### 4.3 Result Variation

作者实验发现，`R`的选择很重要，上述实验结果是在`R=0.2`下得出的。如下是不同的`R`的实验结果。

![](/images/2019-11-22/Untitled-2ec0b1aa-8868-4787-81be-c9faa2000b07.png)

可以看出，`R=0.2`是一个比较好的选择。

### 4.4 假阳性和假阴性

在Linux-3.14上的实验结果里，假阳性比例为5.9%，在Linux-4.19上，假阳性比例为5.7%。这些假阳性结果，作者发现一些原因是：

1. Lockset Analysis中的别名分析，无法区分存于同一个结构体域的不同变量，会认为都是相同的。
2. Lockset Analysis是流敏感，但是不是分支敏感的。
3. Lockset Analysis只检测跟锁相关的函数调用，并不检测其他形式的同步。

对于假阴性结果，作者发现一些原因包括：

1. Local-global策略和Lockset Analysis里缺少对函数指针的分析，因此无法创建完整的调用关系图 (Call Graph)。
2. 别名分析有缺陷，在函数参数和指针赋值处会出问题，会将两个一致的变量认为是不同的变量。
3. Local-global策略里忽视了一些真实存在的case，例如不考虑驱动函数会与自己并发。
4. RCU的同步方式，函数调用不需要参数，因此DCUAF不好识别。
