---
layout: post
title: "UBITect: A Precise and Scalable Method to Detect Use-Before-Initialization Bugs in Linux Kernel"
date: 2020-10-15 15:31:50 +0800
comments: true
categories: 
---

> 作者：Yizhuo Zhai, Yu Hao, Hang Zhang, Daimeng Wang, Chengyu Song, Zhiyun Qian, Mohsen Lesani, Srikanth V. Krishnamurthy, Paul Yu
> 单位：UC, Riverside
> 出处：ACM SIGSOFT Symposium on the Foundation of Software Engineering/ European Software Engineering Conference (FSE 2020)
> 原文：https://www.cs.ucr.edu/~csong/fse20-ubitect.pdf

## Abstract

未初始化使用（Use-Before-Initialization，UBI）bug在linux kernel中会造成严重的安全影响，包括信息泄露和权限提升。由于难以推断正确的初始化值，即使代码进行强制初始化，也会造成空指针解引用等问题。
对UBI bug精确的检测需要路径敏感的分析，记录所有的变量初始化状态以及不同执行路径上的使用。但是这种分析无法应用到代码量极大的linux kernel上。
本文作者实现了一种工具UBITECT，将流敏感的类型限定词分析与符号执行结合以进行精确和可扩展的UBI bug检测。类型限定词分析引导符号执行对可能造成UBI bug的变量进行分析。
对于4.14版本Linux kernel，UBITEST报告了190个bug，其中有78个为true positive，52个被Linux维护者确认。

<!-- more -->

## Introduction

检测UBI bug主要有两个挑战：
1. Linux Kernel有超过2000万行代码，所以分析必须是可扩展的（scalable）
2. 大多数UBI bug是路径敏感的，意味着它们只会在特定路径上被触发，所以需要跨过程的路径敏感分析，与第一个挑战相悖。

为解决上述挑战，作者首先使用了soundy的流敏感，成员变量敏感以及上下文敏感的过程间静态分析来寻找潜在的bug。对于每一个可能的bug，这一阶段会生成符号执行探索路径的引导，跳过变量不被使用的路径以及变量被初始化的路径。
在此基础之上，UBITECT利用under-constrained符号执行来根据引导寻找一条可达路径，并报告bug。
作者针对allyesconfig配置的Linux kernel（16163个文件，616893个函数）对UBITECT进行了评估。工具报告了190个bug，作者手工确认了78个，误报率59%。对于报告准确的bug，作者发现9个已经在kernel中被移除，11个已经被修复。作者提交了剩下58个bug的patch，其中37个被接受。
此外，作者利用启发式的方法，额外发现了15个bug。

作者贡献：
1. 将可扩展的符号限定词推断与符号执行结合，对Linux kernel进行了可扩展高精度的未初始化使用bug进行了检测
2. 基于LLVM与KLEE实现了UBITECT，共13446行代码
3. 发现了78个bug，除去11个已被修复，有37个被Linux维护者确认。

## Background

### use-before-initialization bugs
![](/images/2020-10-15/uploads/upload_9bc6274a696b58c6ad2cd10705669da7.png)

通过栈喷以控制未初始化的函数指针，达到控制流劫持的目的。

### Challenges in Detecting UBI Bugs

![](/images/2020-10-15/uploads/upload_e396806be29fd02dfa041dc56644f8ad.png)

* 需要过程间分析以及路径敏感分析
    保守的过程内分析需要所有的变量均被初始化，包括指针以及指针指向的数据。当上述变量以指针形式传递给子函数时，由于子函数并不会对所有的变量进行读取，所以在一定程度上会引起误报。以上图为例，传递进函数的指针是否被初始化，取决于函数是否会进行错误处理（15行）。此外由于函数并不会对参数所有的成员变量进行访问，所以域敏感也是必要的。
    
## Overview

![](/images/2020-10-15/uploads/upload_5d9909ed5adcc2bf6b6ce45afe396b48.png)

### Pre-processing
UBITECT首先将kernel编译为LLVM IR。为了提高类型限定词的扩展性，UBITECT使用了自底向上的过程间分析。
作者对全程序的间接调用进行了分析，构建了caller与callee之间的依赖关系，在此基础上对kernel全代码生成了call graph。
此外，作者也用call graph将可能存在的递归进行记录。

### 类型限定词推断
此前的工作已经将类型限定词推断应用到安全漏洞检测，例如利用kernel和user限定词来追踪用户空间可控的内核指针以及寻找用户提供指针的不安全解引用。
作者新定义了init和uninit两个限定词，并且init是uninit的子集，以及基于此的init int等。
* uninit的表达式不能被赋值到init的位置（location）上；
* 只有init类型的表达式可以被解引用；
* 只有init类型的表达式可以用于分支跳转。

UBITECT会在函数的每一个程序点的每一个变量赋上限定词，并自动进行推断，如果推断可以正确进行，则函数不会存在UBI漏洞。否则UBITECT会根据assert为符号执行生成引导。具体方法为为每一个LLVM Value设置一个符号类型，对于不同类型的Value，UBITECT会生成不同的子类型约束。遇到安全相关操作时，UBITECT确保相关表达式具有限定词init（通过约束求解计算）。



#### Function Summaries Generations
上述过程结束后，针对每一个函数，UBITECT会生成函数概要（function summary），包括：
1. **qualifier requirement**：表示函数不会触发UBI bug的输入参数限定词要求
2. **qualifier update**：表示输入输出参数的限定词更新
3. 返回值的限定词

以图4为例，第五行的ns_name和ns_len参数之后会被解引用，所以限定词推断会为它们分别赋值为`k1 const char k2* k3*`以及`k4 size_t k5*`, UBITECT可以推导出来它们的限定词至少为 `uninit const char uninit* init*`和`uninit size_t init*`. 进一步分析，在第九行和第十行会将两个对象更新为`k1 const char init* init*`和`init size_t init*`。但是由于在另一分支不会被更新，所以在函数返回时，会将k2和k4标记为`k2 v init`和`k4 v init`

在上述基础上会进行过程间分析。
![](/images/2020-10-15/uploads/upload_72c4217481a4c0d172c651cb49d92eae.png)

#### Guidance for Symbolic Execution

* avoidlist
    * 变量被初始化
    * 基本块内没有对变量的使用
* mustlist
    * 变量未被初始化
    * 未初始化变量被使用

## Design

### Points-to and Aliasing Analysis
流敏感和成员变量敏感的过程内指向分析以及标准的数据流分析。对于每一条语句，一个point-to map会根据控制流来更新。同一个指针在不同程序点的指向集合不同。
由于类型转换在内核中很常见，UBITECT会追踪所有的变量和对象而无视它们是否为指针

### Qualifier Inference

公式较多，理论性比较强，跳过吧...

### Inter-Procedural Analysis
对于call graph中的函数F，类型推断会对其子函数分别进行，并生成函数概要。QR表示函数不会触发UBI bug最弱的限定词条件。QU记录了输出的限定词变化（主要针对输入的指针）。并且为了支持上下文敏感和过程间分析，限定词更新和返回值是多态的，根据不同的上下文变化。

限定词要求可以直接从函数中获取，如上图4.
对于限定词更新，UBITECT维护了函数的指针参数在函数入口和出口的别名集合，如果别名集合发生变化，相关参数的限定词就会更新，最终的输出是参数集合的最小上边界。

作者用上述函数概要实现了上下文敏感的过程间分析：
* Inference constraints： 函数的实际参数必须是形式参数的子集。这样可以保证函数被安全的调用。
* Apply updates: 函数调用之后，指针类型参数的指向集合的值的限定词会被更新
* 对于间接调用，函数的参数必须满足所有可能调用函数的限定词要求，并且更新时也会保守的选取所有可能调用函数的限定词更新的最小上边界。

### Symbolic Execution
在过程间的类型限定词推断之后，符号执行将对报告的漏洞点进行分析。
符号执行模块会首先将bug相关的bitcode进行链接，并从未初始化变量定义的函数开始探索。根据avoidlist和mustlist的记录，符号执行模块会对路径进行选择。最终可以保证true positive不会被过滤。

## Implementation

UBITECT基于LLVM实现，采用了KINT进行全内核分析并修改以适配自底向上分析。指针分析基于Andersen’s inclusion-based pointer analysis。符号执行选取了klee。
作者对三大类共26个函数进行了人工处理，包括LLVM builtin函数，堆分配函数以及安全相关函数。

## Evaluation

作者的实验主要回答以下三个research question：
1. Can UbiTect detect previously known bugs?
2. Can UbiTect detect new bugs?
3. Compared with UbiTect, how do other open sourced static analyzers perform for finding UBI bugs in the Linux kernel?

**实验环境**
Intel(R) Xeon(R) E5-2695v4 processors and 256GB RAM
64 bit Ubuntu 16.04 LTS

### Detecting Known UBI Bugs
![](/images/2020-10-15/uploads/upload_038a8761e707858a42e92aea8991138c.png)

### Detecting New UBI Bugs
UBITECT用了一周完成了全内核616893个函数的分析。7天的CPU时间用于限定词推断，205天的CPU时间用于符号执行。
限定词推断生成了147643个栈上变量的未初始化使用报告，包含了对同一变量在不同语句上的访问以及同一对象不同成员变量的访问。
**由于作者对堆上变量的建模过于保守，以及栈上变量的warning已经足够多，所以作者除去了对堆上变量写未初始化值的warning。**

符号执行模块过滤了4150个路径不可达的warning，另外有1190个warning无法处理，因为存在32和64位指针混用的情况。
作者手工选取了190个两分钟之内找到可达路径的bug，其中6个与未初始化函数指针相关，125个与未初始化数据指针相关，59个与未初始化可影响控制流的数据相关。
对于112个误报，作者也将其进行了分类，包括：1）过程间函数较多时，黑白名单的引导不够准确；2）间接调用结果不准确；3）LLVM优化导致的函数概要不准确；4）UCSE的限制；5）汇编代码。
![](/images/2020-10-15/uploads/upload_f33d5d112b29cbf82bd4385449a932ca.png)

### Comparison with other Static Analyzers

作者将UBITECT与cppcheck以及CSA进行了对比。
cppcheck报告了164个bug，其中2个为TP。
作者使用CSA对78个UBITECT确认bug的文件进行了测试，花费了96分钟，检测到17个真实漏洞以及5个误报。但是CSA是过程间分析，对全内核的分析结果会更差。