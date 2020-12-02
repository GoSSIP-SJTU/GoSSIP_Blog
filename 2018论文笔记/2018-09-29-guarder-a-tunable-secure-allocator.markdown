---
layout: post
title: "Guarder: A Tunable Secure Allocator"
date: 2018-09-29 18:13:49 +0800
comments: true
categories: 
---

**会议**：USENIX Security  2018

**作者**：Sam Silvestro,  Hongyu Liu, Tianyi Liu, Zhiqiang Lin, Tongping Liu

**原文**：[silvestro.pdf](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-silvestro.pdf)

---

### 背景简介

linux系统默认的堆分配器ptmalloc自设计之初就未考虑到安全性问题，因此关于堆攻击的问题被研究得很多。由于设计理念上的缺陷，ptmalloc的安全性问题不能仅靠打几个补丁来解决，因此许多关于安全的堆分配器方案被提出来，这篇论文就提出了一个安全的堆分配器方案guarder。这是林志强老师在安全的堆分配器上的第二次实践，他的上一篇论文提出的free guarder也是一个安全堆分配器方案，在这篇论文里面，作者多次提到FreeGuard并与之作出比较并得出相对于FreeGuard,Guarder更加安全的结论。

<!--more-->

##### 堆相关漏洞

double free, use-after-free, heap overflow, invalid-free, heap-overread.

##### 根本解决方法之一
1. 使得堆分配地址不可预测，即攻击者在得知当前堆状态的情况下无法预测下次堆分配出的地址位置，即堆分配的熵值。
2. 在堆分配的内存对象之间放置无效页面，当发生堆溢出时即可触发页访问保护，即可检测堆溢出
3. 在堆分配内存对象后放置canary检测堆溢出。
4. 将元数据与用户数据分开存放，使得发生溢出时不会损坏元数据。

### 系统设计

![](/images/2018-09-29/1.png)

上图为该系统的总体设计情况。guarder首先分配一大块内存，将这块内存分成m部分，每个部分对应不同线程用到的堆内存T。在每个T内部再按大小将内存分为16B至256B的小块(bag)。这些内存的大小都为2的n次方。每个bag对应了一个alloc buffer和一个dealloc buffer。alloc buffer中记录了当前bag中可分配的内存。dealloc buffer中临时记录了在该线程中释放对应大小的内存。在每个bag内部有相同大小的内存对象obj，在obj之间放置了guard page用来检测堆溢出。

##### alloc buffer
alloc buffer用来记录每个bag中可用的内存块，记录的内容为该块内存的地址而并非索引。alloc buffer的大小是可控的，用户可以根据自己的安全需要定制不同的大小，并且每个bag所对应alloc buffer大小不受其他bag的影响。假设用户定义的堆熵值为E，那么对应的alloc buffer的大小为2E。
在分配内存时堆分配器随机的从这2E个内存对象中抽取一个返回给程序使用，随着程序的运行，alloc buffer中可用的内存对象在减少，当alloc buffer的大小下降到E时则会从bag中抽取新的内存对象补充到对应的alloc buffer中以保证攻击者猜中堆地址的概率小于1/E。

##### dealloc buffer
dealloc buffer用来临时记录当前线程释放的内存对象，是一个环状缓冲区。注意guarder中线程释放的对象归还到当前线程的dealloc buffer中而不是申请内存所在的线程堆中。当发生内存分配后，dealloc buffer中的内存对象被加入到alloc buffer中用于下次分配。


##### over-provisional
每个bag中的一部分内存对象是不被使用的，用户可以自由配置该不可用对象对占的比例，在向alloc buffer中添加新的bag中的内存对象时，根据用户定义的比例选择是否添加该对象，这种作法同样提高了熵值。

##### guard page
每个bag中的内存对象中间随机的放置guard page。作用是在堆溢出时能通过页面访问控制捕获到这种异常。这个比例也是用户可控的。

##### canary
在用户分配所需要的内存的末尾放置一个字节的canary，在释放内存时检查canary是否完好，以此来检测堆溢出。

##### 大内存对象
guarder将大于512KB的内存对象视为大内存对象，作法是直接使用mmap进行内存分配。

##### meta data
meta data被隔离放置，并且meta data的内存是动态申请的。meta data中存储着堆内存对象的状态。在使用中还是被释放。

##### 优化
为了优化内存消耗和提高性能，作者使用了以下几种方法：
1. 使用栈内存地址来查询当前线程ID
2. 按需初始化alloc buffer
3. 当大于64KB的内存块被释放时，使用madvise的MADV_DONTNEED释放该内存


### 设计评估
通过从alloc buffer中随机选择内存对象，over-provisioning这两种方法可以保证内存对象分配的不可预测性，从而使得use-after-free难以利用。另外隔离meta data，guard page可以使得堆溢出能被检测出来。


### 评估

##### 测试软件，硬件，系统环境
 - 硬件： IntelXeon E5-2640 256G内存
 - 系统环境： Linux4.4.25，编译器：gcc-4.9.1 -o2 -g
 - 测试软件： 见下表
  ![](/images/2018-09-29/2.png)

除guarder之外，作者另外选取了TCMalloc,DieHarder,OpenBSD,FreeGuard作为比较对象，表中竖状图y坐标含义为，以ptmalloc为基准1，当数值大于1时表示性能较差，数值小1时表示性能好。横坐标为不同的测试软件。对于guarder，作者选择的配置参数为
1. 9bit的buffer熵值，即alloc buffer的初始大小为2**10
2. 10%的随机guard page
3. 1/8的over-provisioning

从图表中可以看出，guarder和freeguard的性能均略微比ptmalloc要差，DieHarder和openbsd性能比较差，分别为74%，31%。而专注于性能的TCMalloc则比ptmalloc性能要好。

#####当配置参数变化时guarder的性能变化如下图所示

![](/images/2018-09-29/3.png)

作者使用控制变量法来进行测试，可以发现，随着安全性的提升，guarder的性能呈下降趋势，而这也是作者所强调的，可配置的安全性策略堆分配器。即以性能换安全性。

##### 性能分析

###### 系统调用所带来的性能开销
由于guarder初始化时通过mmap申请大量内存，因此mmap的使用次数较少。放置guard page时需要大量使用mprotect，但是作者发现mmap系统调用的时间是mprotect的20倍。因此相对来说可以抵销一部分性能开销。

###### 堆分配开销
堆分配开销主要来源为每次分配所需要的查找次数
ptmalloc, TCMalloc只需要查找一次。guarder平均需要1.77次而open BSD和DieHarder分别需要3.79和1.99次。
对于内存释放操作，DieHarder平均需要12.4次，而openBSD,FreeGuard,GUARDER只需要一次。
由于GUARDER每次释放内存时均将内存归还到当前线程的dealloc buffer中因此不需要上锁，所以这里没有产生线程同步的性能开销。

###### 内存消耗
对于标准配置的GUARDER，内存消耗平均为27%而FreeGuard平均为27%。DieHarder，OpenBSD分别比ptmalloc少消耗内存3%和6%。从这里可以看出GUARDER和FreeGuard所带来的安全性是以较高的内存消耗换来的。

### 评价
GUARDER相对FreeGuarder来说性能和内存消耗上并没有太高的进步，对于安全性来说也还有待考量。对于堆分配的不可预测性，在实践中可能需要多次堆地址的泄漏才能完成一次攻击，而每一次的堆地址都是不可预测的，这样即每一次的不可预测性并不是很高，但是整体带的结果也是不可忽视的。另外对于这种安全的堆分配器十分依赖于大的虚拟地址空间，即他们并不会对内存对象块进行切分，导致分配一次即分配个页面，这样使得虚拟内存空间消耗很大，这是安全的内存分配器的通病。另外作者提到的canary，仅有一个byte，这样一个字节的canary溢出检测是否不太严谨。
安全的堆分配器有一天会成为堆的标准，因此是一个很有潜力的研究方向。
