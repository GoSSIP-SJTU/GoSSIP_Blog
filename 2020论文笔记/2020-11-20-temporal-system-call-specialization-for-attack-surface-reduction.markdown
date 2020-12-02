---
layout: post
title: "Temporal System Call Specialization for Attack Surface Reduction"
date: 2020-11-20 15:42:22 +0800
comments: true
categories: 
---

> 作者：Seyedhamed Ghavamnia, Tapti Palit, Shachee Mishra, Michalis Polychronakis
> 
> 单位：Stony Brook University
> 
> 出处：USENIX Security 2020
> 
> 原文：[https://www.usenix.org/system/files/sec20-ghavamnia.pdf](https://www.usenix.org/system/files/sec20-ghavamnia.pdf)

## Introduction

通过移除应用程序不必要的组件或者代码，可以在不引入额外开销的情况下降低攻击面。
之前已经有类似的工作，例如usenix security 2018的[Piece-Wise](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-quach.pdf)以及ACSAC 2019的[Nibbler](https://cs.brown.edu/~vpk/papers/nibbler.acsac19.pdf). 这两个工作分别针对源码和二进制文件进行了library debloating。但是这两个工作共有的一个缺点是他们将程序运行的生命期当作一个整体进行分析。实际上程序运行不同阶段的行为和调用的syscall均有较大差别，分阶段禁用syscall可以为减少攻击面带来更好的效果。

在本文中，作者提出了一种名为temporal specialization的新方法，通过在代码的不同执行阶段限制可用的syscall集合以减少攻击面。作者发现多数服务端代码会分为初始阶段和服务阶段（initialization and serving phases），并且一些较为敏感的syscall（类似execve）可以在初始阶段之后被禁用。
作者基于LLVM实现了方法的原型，并选取了6个流行的开源应用进行了评估。
结果显示作者的方法可以比现有的library specialization方法多禁用51%的syscall，且可以抵御13个Linux的提权漏洞。

分析程序不同阶段使用的syscall，需要通过构建可靠和准确的call graph。但是对于c和c++编写的程序，由于函数指针的存在，分析会变得比较复杂。即使是最先进的指向分析也会收到不精确和过估计的影响。作者利用类型信息和可达性信息提高了指向分析的准确度。
在对不同阶段syscall的使用分析完成后，作者用Seccomp BPF禁用了不需要的syscall，达到了降低攻击面的目的。

<!-- more -->

## Background, Motivation and Threat Model
![](/images/2020-11-20/uploads/upload_f55282d01ec13423bd20578dd1c694d1.png)

从上图可以看出Apache服务在启动阶段和服务阶段调用函数的区别。

### Seccomp BPF

Linux kernel可以利用Seccomp BPF（Berkeley Packet Filter）机制来限制用户态程序可用syscall的集合，提供了allow，log，deny等策略。可以通过prctl或者seccomp系统调用来设置seccomp。设置之后进程或者进程fork的子进程都会收到seccomp的影响。

### Threat Model

本文的威胁模型是攻击者具有任意代码执行的能力。作者的方法限制了攻击者调用syscall的能力，从而使对应的利用代码（shellcode或者ROP）失效或者能力被限制。

## Design and Implementation

![](/images/2020-11-20/uploads/upload_84fd1180058f327ff994748e4b4ae5b5.png)

作者的目标是降低初始阶段之后暴露给攻击者syscall的数量。方法的总体流程如上图所示，包括以下几个步骤：

1. 构建程序call graph，定位三方库导入的外部函数
2. 将外部函数以及call graph与syscall进行对应
3. **手工**寻找程序初始化阶段和服务阶段开始的标识，并以此将call graph划分为不同阶段
4. 基于call graph，识别不同阶段使用的syscall
5. 基于上述信息创建seccomp filter进行过滤

### Identifying the Transition Point

Apache Httpd和Nginx：主进程fork，子进程被创建
Memcached：事件驱动模型，分界点是event loop的开始
作者表示，可以实现自动化寻找分界点，但是没有必要。

### Call Graph Construction

这里是作者设计的主要部分。定位不同阶段使用的syscall需要对程序生成准确的call graph。但是c或者c++的程序中存在大量的函数指针和间接调用，会使分析的难度增加。
作者使用SVF框架的point-to analysis，可以获取程序的完备的call graph（即可以正确识别出所有的间接跳转，但同时会包含大量误报）。
以Apache的ap_run_pre_config函数为例，SVF会报告136个间接跳转的目标，但是作者验证之后发现7个是正确的目标。

#### Points-to Analysis Overapproximation
程序分析中的context-sensitive和path-sensitive会对分析产生很大影响，实际分析时会在性能和精度之间折衷。
context-sensitive意味着分析会区分不同callsite的上下文，进而对传入参数和返回值进行区分。缺失上下文敏感会导致单个callsite的参数和返回值会传播到对应函数所有的callsite。
path-sensitive意味着分析可以区分不同的分支跳转。例如libapr支持注册一系列回调函数，并在实际执行时通过判断函数指针是否为空来判断是否执行回调函数。

指向分析时会默认所有路径都有执行的可能，但实际并不如此。服务端代码通过函数注册的方式来对间接调用的函数指针进行赋值，即使函数没有被注册（即程序运行时函数指针并不指向相关函数），指向分析还是会认为有指向的可能，造成了过估计。作者使用了两个启发式的方法来解决此问题。

![](/images/2020-11-20/uploads/upload_e4c8d442f06be5f927a0664a0e4d37a9.png)


#### Pruning Based on Argument Types

函数指针指向的不同函数应该与函数指针具有相同的参数类型，所以可以根据参数类型对指向集合进行过滤。这里作者只考虑了结构体类型的参数以达到可靠性的保证。针对Nginx，此方法过滤了call graph中间接调用70%的边。

#### Pruning Based on Taken Addresses

当一个函数的地址被赋值给一个函数指针，一般有以下三种情况：

1. 直接存储到一个指针中
2. 当作参数传递给另一个函数
3. 作为全局变量的初始化

作者表示，只有代码中存在相关指令时，才表示函数可能作为一个函数指针的指向对象。通过收集代码中相关指令的信息，作者进一步对函数指针的指向集合做了过滤。

此外，只有相关指令所在的函数可以从main函数到达时，作者才认为对应的间接调用

![](/images/2020-11-20/uploads/upload_495d2066ed4130b17305bc758fe2a56a.png)


### Pinpointing System Call Invocations

用户程序调用syscall的方式一般是通过libc，最常用的是glibc。由于glibc无法利用LLVM编译，所以作者利用RTL IR对libc的函数进行了分析。
大致思路也是构建call graph，将syscall当作图的叶子结点。
作者也对内联汇编等更复杂的场景做了一些描述。

### Installing Seccomp Filters
手动实现

## Evaluation

作者选取了六个流行的开源应用Nginx, Apache Httpd, Lighttpd, Bind, Memcached, 以及Redis进行测试。
为了进行对比，作者也基于SVF实现了library debloating，并进行了一系列实验进行论证。
对于temporal specialization正确性的验证，作者选择对每一个应用发送100个请求并验证请求的响应。

作者通过各种途径获取了567个shellcode，并基于此生成了1726个shellcode的变种。

![](/images/2020-11-20/uploads/upload_1b01127f9f4c7c514eba3cff4c3f622d.png)
![](/images/2020-11-20/uploads/upload_e0279c4f276ce36ca03f7592637a435b.png)
![](/images/2020-11-20/uploads/upload_6c7e51ce091fe5f894795db49b51c38f.png)
![](/images/2020-11-20/uploads/upload_7ea2dff2db5b182b9025522565917845.png)
![](/images/2020-11-20/uploads/upload_38b4b64012ce72667f8871e04a4cc1c8.png)
![](/images/2020-11-20/uploads/upload_95a498ab19261cd0c65828bd6c9f9c33.png)
