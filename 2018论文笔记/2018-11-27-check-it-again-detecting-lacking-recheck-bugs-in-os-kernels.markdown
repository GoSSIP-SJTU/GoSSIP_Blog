---
layout: post
title: "Check It Again: Detecting Lacking-Recheck Bugs in OS Kernels"
date: 2018-11-27 16:52:04 +0800
comments: true
categories: 
---

作者：Wenwen Wang, Kangjie Lu, and Pen-Chung Yew

单位：University of Minnesota

出处：ACM CCS'18

原文：https://www-users.cs.umn.edu/~kjlu/papers/lrsan.pdf



## 1. 简介

论文中，作者主要介绍分析操作系统内核中的LRC（lacking-recheck）类型bug。并介绍了自己设计的一个静态分析系统LRSan（在Linux上实现），用于检测操作系统内核中的LRC bug。

本文的主要贡献：
1. 定义了LRC bugs，并第一次展示了关于LRC bug的深入研究。
2. 实现一个自动化的LRC bug检测系统（LRSan），基于LLVM，使用在Linux内核上，可以检测内核中的LRC bug。LRSan运用了很多新的程序静态分析技术。结果显示LRSan在Linux内核发现了2808个LRC case，检测耗时为4小时。并且作者会将它开源。
3. 识别**Security check**（*SC*）和相关**Critical variable**（*CV*）的方法。
4. 发现了Linux内核中的19个新的LRC bug。

<!--more-->
## 2. LRC Bugs

如下图所示是LRC Bug形成的流程。

![-w443](/images/2018-11-27/media/15414989347031/15417485806882.jpg)

如下图所示是一个LRC Bug的例子，其中的val就是*CV*。在第10行中val被进行了security check，但是在第16至19行中val被进行了修改，在第22行中val又被使用了，然而在使用前没有进行recheck。

![-w476](/images/2018-11-27/media/15414989347031/15417476996073.jpg)

LRC bug的特点：
- 拥有一个check-use chain。即执行路径中有对变量进行安全检查，并在之后对变量进行使用。
- 变量有被修改的可能. 变量在过了前面的security check后，在check-use chain中被修改。
- 缺少recheck。在变量过了安全检查后，如果在使用前又进行了安全检查，那即是在中间被修改了，也是安全的。因此缺少recheck也是LRC bug的成因之一。

作者在文章中还具体符号化定义了Security check、Use、Modification、Lacking recheck。

### 2.1 Security check

Security check需要满足两个条件（作者通过观察总结的规律）才能被认定为是Security check：
- 拥有一个条件语句，紧随其后的是两个分支：
    1. 一个分支肯定会返回error code（condition 1）
    2. 另一个分支有可能不返回error code（condition 2）

### 2.2 Modification

在执行路径上，通常变量的的security check和使用是比较复杂也可能比较远的，尤其是在处理涉及到用户空间和内核空间的多个变量时，因此LRC Bug在操作系统内核中是很常见的。

对security-checked变量的修改可能由如下一些情况导致：
1. Kernel race：操作系统内核通常为线程进程维护了很多共享的数据结构。通常来说很难保证Security checked variable不被修改。
2. User race：从用户空间抓取数据通常很容易被通过用户空间的多进程来进行修改。
3. Logic errors：一些逻辑问题可能导致线程本身不正确地将security-checked variable给修改了。
4. Semantic errors：例如类型转换和整数溢出的一些语义错误可能导致security-checked variable被修改。

## 3. LRSan设计

LRSan的整个结构和Workflow如下图所示，输入为内核源码编译后的LLVM IR。LRSan预处理时会构建一个全局的call-graph，用来进行inter-procedural analysis，并且标注了error codes。

![](/images/2018-11-27/media/15414989347031/15417508986947.jpg)

LRsan设计的4个关键部分：
1. Security check identification
2. Critical variable inference
3. check-use chain construction
4. modification inference。

### 3.1 Automated Security Check Identification

作者将security check认为是一个条件语句（例如if语句）。因此检测Secruity check就是查找这样的条件语句。

1. 搜集所有的error code，构建一个error-code CFG（ECFG），ECFG可以快速地判断出某一个执行路径是否会返回error code。ECFG的示例如下图所示。
2. 通过前面提到的两个判断Security check的条件，来决定找到的条件语句是否是Security check

![](/images/2018-11-27/media/15414989347031/15431388837297.jpg)

### 3.2 Recursive Critical Variable Inference

在前面的步骤中可以得到一个Security check set，通过这个set能直接得到Critcal variable set(CSet)，然后再通过CSet去找到更多的*CV*，查找的原则是越多越好（更少的假阴性，这里用的是反向的污点分析）。

例如下图，hdr是最先找到的*CV*，然后又将arg和buf都标记为*CV*。

![](/images/2018-11-27/media/15414989347031/15431437250915.jpg)

查找终止的条件是在寻找的过程中中碰到了**hard-to-track values**，例如全局变量，堆上的对象，用户空间的对象，这些可能是来自shared data，external source，如果把这些也标记上，会使得查找更加困难。

### 3.3 Check-Use Chain Construction

这一步为*CSet*中的每个变量构建**Check-use chain**。**Check-use chain**可以用一个四元组来表示<*SC, CV , Iu , PSet*>。

*Iu*：*CV*在*SC*后的第一次使用
*PSet*：*SC*后的一系列执行路径（到*Iu*结束）

构建**Check-use chain**的方法：污点追踪找到*Iu*，遍历**CFG**去收集一系列从*SC*开始，*Iu*结束的执行路径，这些执行路径的结果记为*PSet*。

这里还使用了LLVM里的Alias analysis来寻找一些隐式的use
### 3.4 Modification Analysis

在找到了**Check-use chain** <*SC, CV , I u , PSet*>后，这一步的目的是识别潜在的对*CV*的修改（**Check-use chain**里找）。

主要检测的修改操作：
1. *CV*被修改成了一个新的值，通过一个普通的store指令
2. *CV*被例如memcpy或者copy_from_user等函数进行了修改。

如果找到了这样的修改，LRSan会再进一步检测这次修改之后是否有有critical variable进行recheck。最后将没有recheck的情况就认定为是LRC case。

## 4. LRSan实现

作者基于LLVM 6.0.0实现了LRSan。整个实现中包括两个独立的pass（大约3000行代码）。First pass是用来收集和准备静态分析锁需要的信息，例如构建global CFG，alias results。Second pass是用来进行静态分析检测LRC case的。

作者成功编译了16593个module，只有6个编译失败了。

过滤部分false positive：
1. *CV*被修改成的是一个常量
2. *CV*被修改成的是一个可以过Security check的变量
3. mutex-style check

## 5. 评估

### 5.1 Effectiveness

如下图所示是LRSan检测后导出的检测结果。

![-w610](/images/2018-11-27/media/15414989347031/15432813596211.jpg)

如下图所示，是LRSan在Linux kernel里找到的19个LRC bug。

![](/images/2018-11-27/media/15414989347031/15417520389602.jpg)

### 5.2 Efficiency

LRSan测试时所用的LLVM IR是用Linux-4.17的源码编译的。作者实验时用的host为一个主存32G的Ubuntu16.04，Quad-Core 3.5 GHz Intel Xeon E5-1620 v4处理器，总共用了4个小时。其中超过80%的时间都是用在first pass（例如信息收集），因为first pass需要构建一个全局的CFG，并收集alias-analysis results。first pass实际上只需要运行一次，因此可以对它的结果进行重用，从而节省时间。

## 6. Limitation

### 6.1 False Positives

1. Checked modification，符号执行来过滤
2. Satisfiable modification，符号执行来过滤
3. Uncontrollable modification，污点追踪来过滤
4. Transient check，作者手工排查
5. Unconfirmed race：source variable是一个shared variable（例如全局变量），即有可能被其他的线程进程修改，这是作者未来的工作
6. Other：静态分析的一些缺陷导致的false positives

 ![-w704](/images/2018-11-27/media/15414989347031/15432806874260.jpg)

### 6.2 False Negatives

作者在查找security check时是基于失败后会返回error code，但是也有可能不返回error code。另外编译失败的一些kernel module也可能导致False negatives。
