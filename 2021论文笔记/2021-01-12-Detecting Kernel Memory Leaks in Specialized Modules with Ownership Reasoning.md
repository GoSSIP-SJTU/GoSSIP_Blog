# Detecting Kernel Memory Leaks in Specialized Modules with Ownership Reasoning

> 作者：Navid Emamdoost, Qiushi Wu, Kangjie Lu, Stephen McCamant
> 
> 单位：University of Minnesota
> 
> 会议：NDSS'2021
> 
> 论文链接：https://www-users.cs.umn.edu/~kjlu/papers/k-meld.pdf

## Abstract

作者实现了K-MELD,使用了所有权推断（ownership reasoning）的方法对内核特定模块的内存泄露漏洞进行了检测。作者利用K-MELD在Linux kernel中检测到218个新的内存泄露bug，包括41个CVE。

<!-- more -->

## Introduction
内存泄露是指动态分配的内存对象在其生命周期结束之后没有进行释放。内核空间的内存泄露比用户空间的危害更大，未正确释放的内存直至系统重启之前均无法使用，攻击者可以利用其进行拒绝服务攻击。

针对内核代码的内存泄露检测面临以下两个挑战：
1. 内核不同模块会有其定制的内存管理函数，准确的检测需要对这些函数进行识别；
2. 内核中指针的传递涉及复杂且冗长的数据流，准确的检测需要准确推断变量的生命周期以及释放点。

现有的方法无法很好的解决上述挑战，造成了检测效果的低下。对于上述挑战，本文作者分别针对**内存管理函数识别**和**内存对象所有权推测**提出了新的方法，在此基础上使内核内存泄露达到了较好的效果。

本文的贡献如下：
1. 开发了一种识别定制化内存分配函数的方法
2. 利用规则挖掘识别分配函数对应释放函数的方法
3. 针对内核对象的所有权推测方法
4. 基于上述实现的K-MELD工具，以及利用工具检测出新的bug

## Design and Implementation of K-MELD

作者实现的检测工具K-MELD（Kernel MEmory Leak Detector）架构如下：
![](/images/2021-01-12/16076595543475.jpg)

与常见的源码静态分析工具类似，K-MELD将源码编译为LLVM IR并构建call-graph。
对于内存分配函数的识别，作者观察到大多数分配函数会返回指针并进行null-check，以此为模式并利用额外的规则过滤，得到了内存分配函数的集合。
在此基础上，利用内核模块的error-handling特性，通过{alloc, check, release, return}的模式，在代码的出错处理分支将内存释放函数与分配函数进行对应。
作者分别利用逃逸分析（escape analysis）和消费者检测（consumer detection）来推测内存对象的所有权（即最终负责释放内存的函数）。逃逸分析用来判断内存对象的向上传递（即当前函数不负责对内存释放）；消费者检测用来递归向下推测内存的释放点（即最终“消费”指针的函数）。
最终作者通过模式匹配的方式，判断内存对象在其释放点是否被正确释放，以确认是否存在内存泄露。

### Alloc/Dealloc detection

作者总结了内存分配函数的特征：
1. 返回指针
2. 指针会马上进行null check
3. 指针不是从其他指针上派生出来
4. 指向的对象在使用前会初始化

考虑到内核中的getter函数，特征三用来对分配函数进行过滤。
具体实现时，作者使用了use-finding和source-finding两种方法。前者用来判断指针是否进行null check；后者来检测指针是否从其他指针派生。
另外，特征四并不是所有分配函数的特性，例如kzalloc。

对于与分配函数对应的释放函数，作者考虑到内核的error-handling特性。正常实现的error-handling会在代码运行出错时将系统状态恢复到正常，其中就包括释放相关内存对象。
由于出错处理对应较短的执行路径，所以利用其可以高效的检测与分配函数对应的释放函数。

具体来说，作者使用深度优先的方式来遍历CFG，在过程内收集每个callsite的error handling路径。最终利用<call alloc, check, release, return>的模式进行匹配release函数

### Ownership reasoning

ownership reasoning用来确定最终需要释放内存对象的函数，主要利用了逃逸分析和消费者分析两种技术。

#### escape analysis

某些函数内分配的内存不需要本函数来释放，而是通过返回值，全局变量或者函数参数传递回上层函数。识别类似的函数可以降低实际检测时的误报（即某些没有释放内存的代码点确实是不需要释放的）。

#### consumer analysis

消费者分析与逃逸分析类型，只不过是向下进行分析。本函数分配的内存对象，可能会被其子函数使用，在子函数的出错处理时被释放。识别类似函数可以降低误报和漏报（一方面被子函数释放的内存不能在当前函数再次被释放；另一方法也需要检测子函数中是否对内存进行正确释放）。

实际检测时也需要考虑一些复杂的情况，例如conditional consumer，这里就不做赘述。

![](/images/2021-01-12/16076595770657.jpg)

![](/images/2021-01-12/16076596001661.jpg)

![](/images/2021-01-12/16076596295307.jpg)


### Detecting Bugs Using Mined Rules

作者的方法，在识别分配和释放函数之后就可以进行内存泄露检测。后续的ownership reason是对复杂情况的处理，帮助降低误报。后续的检测也是模式匹配的方式，对于代码中出现的每一个allocation的callsite，作者均利用<alloc, check, release, return>的模式进行匹配，并将不符合模式的callsite进行报告。

## Evaluation

作者的测试针对5.2.13版本的Linux kernel进行，将其编译为allyesconfig版本的bitcode。作者的实验在48核Intel Xeon CPU，256G内存的服务器上进行。

### Scalability

bitcode的生成用了5个小时；内存分配函数的收集用了2个小时；释放函数的识别用了1个小时；对每个分配函数的检查从几秒钟到四小时不等。

### Set of Allocations and Associated Deallocations

作者初始识别出4621个内存分配函数，在释放函数匹配完成之后剩下807个。作者识别的函数可以在这里查看 https://pastebin.com/raw/hBfiqnEp

### Found bugs

K-MELD共产生458个报告，其中240个为误报（52%）。

作者还选取了17个现有的CVE，K-MELD可以检测到其中的9个。无法检测的结果中，有两个是因为API混淆，一个是由于过于复杂的指针传播，还有四个因为版本变更无法复现。

![](/images/2021-01-12/16076596460903.jpg)

作者还进一步测试了escape analysis和consumer analysis的影响，在分别禁用两种分析的情况下，工具报告的数量上升到1292和1386个，且随机抽取新增的报告手工分析均为误报，证明两种方法确实有效。