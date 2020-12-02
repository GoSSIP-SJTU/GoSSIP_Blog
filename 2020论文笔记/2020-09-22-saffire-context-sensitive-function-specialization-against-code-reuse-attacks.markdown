---
layout: post
title: "Saffire: Context-sensitive Function Specialization against Code Reuse Attacks"
date: 2020-09-22 14:20:32 +0800
comments: true
categories: 
---

> 作者：Shachee Mishra, Michalis Polychronakis
> 
> 单位：Stony Brook University
> 
> 出处：EuroS&P 2020
> 
> 资料：[Pa](https://www-users.cs.umn.edu/~kjlu/papers/eecatch.pdf)[per](https://www3.cs.stonybrook.edu/~mikepo/papers/saffire.eurosp20.pdf)

## 1. Abstract

本文作者提出了一个基于编译技术的针对代码重用攻击的防御方案Saffire。对于所有的关键函数函数上下文，Saffire会为它一个函数拷贝，并且严格限制调用时传入的参数。作者用17个真实世界的ROP以及3个完整函数重用的利用进行了评估，Saffire可以成功防御这些攻击，并且引入的开销可以忽略不计。

<!-- more -->

## 2. Introduction & Background

目前代码重用攻击很流行，学术界也已经有很多的针对代码重用攻击的防御措施的工作。除了ASLR和CFI这样的防御措施外，比较流行的方式是移除代码里的不必要的代码，这样做的好处主要有两点：1.减少代码重用攻击中可以重用的代码；2.简化CFI，因为间接跳转少了。

共享库是代码重用的一个主要的目标，因为里面有很多没有被用到的代码。现有的解决方案有库定制 (Library customization)，就是移除程序加载的库中没用到的函数，缺陷是没办法移除一些关键的函数，例如内存分配、进程管理、文件系统操作等，为了解决这个问题就有了API特化 (API specialization)，根据程序实际需要来限制一些关键API函数的接口。[Shredder](https://www3.cs.stonybrook.edu/~mikepo/papers/shredder.acsac18.pdf)通过静态分析Windows二进制应用，对关键函数实施Application-wide Policy来得出限制策略，这有两个主要的缺陷：1) 静态分析函数参数比较复杂 2) 很多参数的值没办法静态确定，只能在运行时才能确定。

本文作者的**目标**就是将API特化中对关键系统函数参数的的限制范围扩大。

本文作者的一个**基本假设**：应用程序用到的系统调用和应用程序代码实际用到的系统调用的语义是不同的。根据应用程序的实际需要特化系统API可以对抗代码重用攻击。

**API特化的挑战：**

1. 二进制 vs 源代码分析：控制流分析和数据流分析都会变困难，本文作者通过在源代码上进行更具体的数据流分析来扩展参数值。
2. 上下文敏感策略：之前的工作不区分不同上下文下函数调用的参数，比如同一个函数在某个上下文下，参数是可以确定的，但是在另一个上下文下参数是不能确定的，那么之前的Application-wide policy会将这个参数认为是不能确定的 (unknown)，如表1所示列出了Nginx里用到的几个libc函数的情况。本文作者解决这个问题的方式是提出的上下文敏感的策略；
3. 动态函数参数：很多函数的参数没办法静态分析出来，比如关于用户输入、环境变量、文件名、内存地址等，目前的一些工作都将这些动态值认为是未知的，因此一些利用仍然可以利用这个缺陷进行攻击，本文作者解决这个问题的方式是提出的Narrow-scope data flow integrity。

![Saffire%20Context-sensitive%20Function%20Specialization%20%20d79658cdded64a0ba50777b6dcf626f6/Untitled.png](/images/2020-09-22/Untitled.png)

**威胁模型：**假设攻击者可以劫持控制流，并且能进行ROP攻击，Data-only attack不在本文考虑范围内。假设ASLR和NX等防御措施都已经部署。Saffire的保护依赖于已有的Software debloating技术，例如Library debloating，因此假设Library debloating已经部署。假设CFI已经部署，这样攻击者不能直接跳过check；假设API checkpointing这样的技术已经部署。

## 3. Function Specialization

函数特化这个环节的目标是创建多个API函数的特化版本，这些特化的函数只能在特定的地方调用。CFI限制了函数只能在合法的Call site调用。Saffire基于CFI，进一步限制了函数在不同上下文下被调用时的参数值。

创建特化版本的库函数需要尽可能地精确识别函数在特定上下文下的参数，作者观察发现参数的值大致可以分为两类：1) 静态，静态分析代码既可确定；2) 动态，只能在运行时确定。例如下图代码所示，ptr和size只有在运行时可以确定，prot则可以在静态分析时就确定，因为是硬编码的。

![Saffire%20Context-sensitive%20Function%20Specialization%20%20d79658cdded64a0ba50777b6dcf626f6/Untitled%201.png](/images/2020-09-22/Untitled%201.png)

作者为静态参数和动态参数设计了不同的处理方式，Static argument binding和Dynamic argument binding。

如下图所示，是Saffire完整的架构图。

![Saffire%20Context-sensitive%20Function%20Specialization%20%20d79658cdded64a0ba50777b6dcf626f6/Untitled%202.png](/images/2020-09-22/Untitled%202.png)

如下图所示是Static binding和Dynamic binding的示例。

![Saffire%20Context-sensitive%20Function%20Specialization%20%20d79658cdded64a0ba50777b6dcf626f6/Untitled%203.png](/images/2020-09-22/Untitled%203.png)

### 3.1 Static Argument Binding

总共三类已知的参数类型：1) 硬编码参数，例如flags、buffer size等；2) Value sets，不同的控制流会设置不同的参数，这些数据中有可以被静态确定的；3) Value ranges，一些特殊的上下文，例如循环里，参数值可以表示为Range。

静态分析中，利用IR的控制流信息来回溯函数的调用者，分析在回溯到main或者找到了静态值结束。

### 3.2 Dynamic Argument Binding

尽管很多参数可以静态决定，但是作者通过实验发现有近乎一般的参数只能在运行时确定。对于这些值，作者将其哈希后的截稿存在Shadow memory里。如下图所示

![Saffire%20Context-sensitive%20Function%20Specialization%20%20d79658cdded64a0ba50777b6dcf626f6/Untitled%204.png](/images/2020-09-22/Untitled%204.png)

Shadow Memory Protection：利用Intel MPK对Shadow memory做进程内的内存隔离。

Multi-threaded Programs: 为每一个线程都创建一个Shadow memory

## 4. Implementation

Saffire的实现是作为LLVM8的Transformation pass形式，在64位Ubuntu19.04上进行测试。Saffire的输入是合并后的bitcode。Saffire对目标API进行保护，通过后向数据流分析实现，

### 4.1 Static Argument Binding

### 4.2 Dynamic Argument Binding

## 5. Evaluation

实验环境：Intel Core i7-6700 CPU, 16GB RAM, 256GB SSD, running 64-bit Ubuntu 19.04.

数据集：1) 从Exploit-DB里挑的17个 Linux-based ROP exploit; 2) Real-world PoC exploit。

总共涉及了11个流行的应用程序，包括服务器 (Nginx、Lighttpd、Vsftpd)，工具(Gzip、OggEnc、OggDec、PuTTY、Ctags)，OpenSSH、Chrome、Firefox

作者从这些Exploit里找出了15个关键的系统库函数（要保护的目标），包括mmap, mmap64, mprotect, open, open64, access, execve, fstat, read, write, clone, pread64, fopen, fwrite, fseek。

### Runtime Overhead

用SPEC2017进行测试，结果如下图所示，平均开销为0.34%。

![Saffire%20Context-sensitive%20Function%20Specialization%20%20d79658cdded64a0ba50777b6dcf626f6/Untitled%205.png](/images/2020-09-22/Untitled%205.png)

### Code and Memory Overhead

如表4所示，对于小型的应用程序，增加的函数比例会比较大，对于大型软件则比较可观。

### Security Evaluation

只有OpenSSH和Lighttpd中不能防住所有的ROP。17个ROP里有6个用execve来获得shell，Lighttpd里有这样的调用，因此无法防住，OpenSSH也是类似的问题。

![Saffire%20Context-sensitive%20Function%20Specialization%20%20d79658cdded64a0ba50777b6dcf626f6/Untitled%206.png](/images/2020-09-22/Untitled%206.png)

![Saffire%20Context-sensitive%20Function%20Specialization%20%20d79658cdded64a0ba50777b6dcf626f6/Untitled%207.png](/images/2020-09-22/Untitled%207.png)