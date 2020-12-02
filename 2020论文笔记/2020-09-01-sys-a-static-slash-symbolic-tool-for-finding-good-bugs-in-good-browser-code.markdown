---
layout: post
title: "Sys: a Static/Symbolic Tool for Finding Good Bugs in Good (Browser) Code"
date: 2020-09-01 13:44:50 +0800
comments: true
categories: 
---

> 作者：Fraser Brown^1, Deian Stefan^2, Dawson Engler^1
> 
> 单位：Stanford University^1, UC San Diego^2
> 
> 出处：29th USENIX Security Symposium (USENIX '20)
> 
> 原文：[Sys: a Static/Symbolic Tool for Finding Good Bugs in Good (Browser) Code](https://cseweb.ucsd.edu/~dstefan/pubs/brown:2020:sys.pdf)

## Abstract

本文介绍了一个将静态检查和符号执行结合起来的漏洞检测工具Sys。Sys将漏洞检测分为两个步骤：首先利用静态检查将可能出现错误的代码进行标记；然后利用符号执行判断标记的代码是否存在bug。
作者利用Sys对Chrome，Firefox以及FreeBSD进行了测试，总共发现51个bug，其中有43个被确认（并且作者获得了很多奖金）。

<!-- more -->

## Introduction

作者的初衷是设计一个可以自动检测浏览器代码漏洞的工具。当前存在一些挑战与问题：

1. **Browsers check a lot**  
    自动生成输入的Fuzzer，运行时检测的sanitizer，传统的静态检测，漏洞悬赏等等
2. **Static checking didn't find much**  
    大型项目一般都会使用静态检测工具，但是上一次静态检测发现高危漏洞还是2014年（作者这么说）
3. **Symbolic execution is hard and slow**  
    全程序符号执行开销巨大，效率太低。对于浏览器或者操作系统这样大规模的代码，基本不可用
    
作者的思路是将静态分析与underconstrained symbolic execution相结合。静态分析用来寻找漏洞模式；符号执行用来确认漏洞，降低误报；underconstrained保证符号执行可以从程序的任意一点开始进行，从而降低符号执行的开销。

作者基于上述思路用Haskell开发了Sys，支持用户自定义static extension和symbolic checker，具有良好的扩展性。

作者的贡献如下：
1. 实现了一个结合静态分析和符号执行的漏洞检测框架，提供了五个checker（包括uninitialized memory, out-of-bounds access, use-after-free以及taint analysis），检测出51个浏览器相关的bug；
2. 提出了将符号执行扩展到大型codebase的方法；
3. 提供了一个基于Haskell，用户可自行扩展的检测系统。

![-w555](/images/2020-09-01/15987254422711.jpg)
    
## System overview

![-w542](/images/2020-09-01/15941900574456.jpg)

为什么这个漏洞没有被现有的工具找到？
1. 需要精确的数值和内存推断，大多数静态工具无法做到这一点
2. 利用测试用例或者动态检测工具触发此bug也是困难的，因为SQLite的代码量过大，到达此bug的路径过于复杂。（需要在Chrome的WebSQL开始，创建特定类型的database，创建virtual table而不是regular table，还需要column是特定的数字才能触发）。
3. 传统的符号执行工具，无法处理此类大规模的codebase。

### How Sys finds the bug

![-w507](/images/2020-09-01/15941910994920.jpg)

1. 静态扫描代码，标记可能的错误点
2. 对标记的location，利用符号执行进行判断
3. 推断Sys遗漏的状态（因为Sys跳过了很多代码）

#### Static

client编写小型的静态分析扩展，用来识别源码中的模式，以便快速扫描代码寻找潜在的错误点。
与传统工具类似，Sys需要先从LLVM IR中构建控制流图，并进行流敏感的图遍历。

Sys与传统静态分析工具有区别。传统的静态分析工具会要求尽量低的误报率，而Sys extension需要的是尽量高的recall（tp/(tp+fn)），即Sys需要尽量减少漏报，而把降低误报的任务交给符号执行。

![-w532](/images/2020-09-01/15941916836054.jpg)

上图介绍了static checker的检测方式。checker本身利用Haskell的`matching syntax(case)`，根据不同的IR指令，执行不同的行为。
checker会首先匹配allocation相关的函数，并将分配的大小x进行记录。
接着，checker会匹配算数运算相关的指令，并将操作数之间的依赖关系进行记录。（例如y=x+1）
x与y之间的依赖给了足够的信息来推测未知状态。（即使不知道x与y的值，但是知道它们之间的关系足以寻找bug）
最终，匹配GEP指令（取索引），如果索引值与最开始内存分配大小相关，则对路径进行标记。


#### Symbolic

static pass将可能存在bug的路径输入给symbolic pass。
symbolic pass会进行两步操作：
1. 对输入的路径进行符号执行
2. 将用户提供的symbolic checker应用到路径上

Sys会探索每个潜在存在bug的路径分支，每个分支都有自己的内存拷贝。沿路径前进时，Sys会将LLVM IR转化为路径约束并求解路径是否可行。

下图是一个用户提供的symbolic checker的示例。首先将输入变量转为符号表示，再利用toBytes方法将偏移转化为字节单位的表示，最后断言arrIndSize大于arrSizeSym（表示一个越界访问）。

![-w592](/images/2020-09-01/15988101867279.jpg)

Checker可以匹配特定的LLVM IR指令，在路径的不同点上面运行，用户可以自行配置。
上述checker会从存在malloc函数调用的函数开头进行符号执行，符号执行的起点也可以选择离main函数更近的位置或者离malloc更近的位置。离main函数越近，得到的信息会越多，开销也会更大。反之，如果直接从malloc开始执行，则会丢失所有的上下文信息，但是具有最小的开销。

## Using Sys to find bugs

### Uninitialized memory

#### static extension
对于每一个栈上分配的内存对象，extension会对allocation后续的所有路径进行流敏感的遍历。如有没有显式的store，extension会将第一个出现的load标记为潜在的未初始化。extension没有追踪指针偏移，而是把每一个偏移都看作是一个新的追踪位置。

#### symbolic chceker
Sys用shadow memory来检测未初始化内存的使用。Sys会对static pass标记的每条路径运行symbolic checker，起始位置是每一个可能未初始化使用的栈变量s。对于s的每一个bit，Sys会在shadow memory里标记一个对应的bit，用1和0表示uninit和not-uninit。对于每一个store，Sys会修改shadow memory里对应的bit为0。在s被读取时，checker会检查shadow memory里对应的bit是否为1，以确认是有存在未初始化内存的使用

![-w368](/images/2020-09-01/15988414897153.jpg)

**false positive**
为了提高效率，checker不会进行被调用的函数（作者说可以通过配置选择进入或者不进入）。如果被跳过的函数刚好是一个内存初始化函数，例如`init(x); *x`，就会导致误报。
为了解决上述问题，Sys对于用作函数参数的指针会增加一个约束，以表示shadow memory里哪些bit需要被置0，简单来说，对于一个没有进入的函数foo(p)，在shadow memory里清空p对应的所有bit（认为跳过的函数都是初始化函数？会有漏报）

#### checker results

![-w429](/images/2020-09-01/15988433498479.jpg)

### Heap out-of-bounds

![-w422](/images/2020-09-01/15988479686339.jpg)


### Concrete out-of-bounds

concrete out-of-bounds是指索引为常量的越界访问

#### static extension

主要对三种操作进行标记：

1. phi节点，可以给操作数引入常量
2. 编译器生成的undef常量，undef可能为任意值，会造成潜在的越界读写
3. 索引为常量的getelementptr指令

static pass会确认上述1，2的常量是否可以到达3，并把这个信息传给symbolic checker。但是pass也会忽略一些情况：比如父类的对象（可能与子类对象的布局不同）以及动态大小的结构体成员变量等等。

#### symbolic checker
由于是对常量的检测，符号执行的作用是过滤不可达路径

## Evaluation

测试平台：Intel Xeon Platinum 8160 (96 threads) with 1TB of RAM, running Arch Linux (2/22/19)
测试对象：Firefox changeset:commithash 503352:8a6afcb74cd9, Chrome commit 0163ca1bd8da, and FreeBSD version 12.0-release

对于Chrome，工具用一个小时完成了OOB检测，六个小时完成了未初始化内存检测。对于FreeBSD，用6分钟完成了用户输入的检测（检测用户可控的输入是否会作为数组索引；是否会作为memcpy等函数的size参数）

### 与其他静态检测工具的对比

![-w414](/images/2020-09-01/15988480120567.jpg)

作者选择了Clang Static Analysis和Semmle与Sys进行对比，进行了未初始化内存的测试。上述两个工具的扩展性足够好，并且已经被应用在Mozilla的代码上。

误报的情况多数是因为静态分析无法对变量的取值进行判断。

![-w426](/images/2020-09-01/15988485885078.jpg)

另外Sys还存在其他工具能检测的漏报，4个是因为Sys跳过了某些函数；4个是因为分析的block超过了Sys的阈值；2个是因为编译器优化消除了bug。作者表示：1）需要考虑未初始化变量作为参数的每个函数；2）对Sys进行优化，增加可以处理的block数量

### 与其他符号执行工具的对比
作者表示angr和KLEE都无法直接对Firefox进行处理。（angr跑了24小时被作者手动停掉）