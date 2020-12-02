---
layout: post
title: "Methodologies for Quantifying (Re-)randomization Security and Timing under JIT-ROP"
date: 2020-11-27 15:45:14 +0800
comments: true
categories: 
---

> 会议：CCS'20
> 
> 论文名称：Methodologies for Quantifying (Re-)randomization Security and Timing under JIT-ROP

# Introduction

linux中的ASLR属于Coarse-grained ASLR，其只会将module/segment作为整体进行随机化，然而攻击者只需要泄露出module中的一个地址，就可以知道该module的基地址，从而可以在ROP时利用整个模块中的gadget。

因此，安全研究人员设计出了多种Fine-grained ASLR，即以functions、basic blocks、instructions、machine register为粒度进行随机化，以缓解ROP类攻击。再后来，又出现了re-randomization的方案，即每间隔一定时间就重新进行随机化，以减小攻击者的时间窗口。

本文提出了一种量化评估上述随机化方案的安全性及re-randomization中间隔时间的方法。

作者的贡献包含以下几个方面：

- 提供了一种测量JIT-ROP gadget可用性、质量和图灵完备的方法
- 提供了一种对于re-randomization间隔时间上界t的计算方法，经过对nginx, proftpd, firefox等应用程序的测试，t的范围在1.5~3.5s。
- 发现了在fine-grained ASLR的场景下，无论用于漏露的起始代码指针在哪里，通过JIT-ROP的方式泄露gadget，泄露出的gadget集合的收敛相同。而起始代码指针的位置与收敛的速度有关。
- 提出了一种量化JIT-ROP gadget数量的通用量化方法。并在实验中发现单轮的instruction-level的随机化方案能够减少约90%可用gadget数量。研究发现相比与堆和data段，更容易通过栈泄露出libc的地址，栈中平均包含16个指向libc的指针。

<!-- more -->

# 传统ROP和JIT-ROP的区别

（假设攻击者可以控制一个指针，可以泄露代码段的数据）为了绕过coarse- and fine-grained ASLR以进行利用，传统ROP和JIT-ROP需要通过以下步骤：

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%201.png](/images/2020-11-27-2/Untitled%201.png)

可以看到，攻击的流程主要分为三个部分：memory layout derandomization, system access, and payload generation。

## Memory Layout Derandomization

### Basic ROP

由于Coarse-grained ASLR只能以module/segment的粒度进行随机化，因此攻击者只需要leak出指向module内部的一个地址即可计算出module的基地址，从而根据预先设置的偏移找到gadgets。

### Just-in-time ROP

Fine-grained ASLR会对code page进行随机化，因此不可能通过线性扫描的方式去探索code page（因为可能读到unmapped memory）。研究人员在2013年JIT-ROP中提出了一种递归下降的code page探索方法，攻击者可以控制一个指向code page的指针，通过这个指针，不断的去检测call jmp指令的跳转地址，从而发现新的code page。如下图所示：

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%202.png](/images/2020-11-27-2/Untitled%202.png)

## System Access

攻击者进行ROP的目的是要获得特权操作，如getshell、文件读写等。因此，通常情况下攻击者需要利用系统API或syscall gadgets组成ROP chain完成利用。由于fine-grained ASLR，JIT-ROP收集gadget的过程比传统ROP繁琐。

## Payload Generation

攻击者将gadgets组合起来，生成ROP chain，将ROP chain存入程序的堆栈、data段，通过stack pivoting等方式触发ROP攻击。

值得注意的是，由于漏洞程序中用于存ROP chain的空间可能有限、构造ROP chain需要花费较大的精力，攻击者通常希望能够构造出“简洁明了”、“副作用较小”的ROP chain。即尽可能使用有且仅有核心功能的，不包含其他更改程序状态指令的gadget。作者将gadget的副作用称为footprint。

# Threat Model

- 实验环境中开启了ASLR：
    - Coarse-grained ASLR
    - Fine-grained ASLR
- 标准的W⊕X 和 RELRO均开启，由于作者关注ASLR这种防御机制的安全性，因此不考虑CFI、Code Pointer Integrity (CPI)
- 假设攻击者已经获得了一个泄露出来的code pointer

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%203.png](/images/2020-11-27-2/Untitled%203.png)

**ASR(USENIX Security'12)**[[paper](https://www.usenix.org/system/files/conference/usenixsecurity12/sec12-final181.pdf)][[slides](https://www.usenix.org/sites/default/files/conference/protected-files/giuffrida_usenixsecurity12_slides.pdf)]:

Function-level

在IR上对代码进行一些变换（函数位置、函数间隔、基本块顺序、结构体字段间隔&顺序...）

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%204.png](/images/2020-11-27-2/Untitled%204.png)

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%205.png](/images/2020-11-27-2/Untitled%205.png)

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%206.png](/images/2020-11-27-2/Untitled%206.png)

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%207.png](/images/2020-11-27-2/Untitled%207.png)

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%208.png](/images/2020-11-27-2/Untitled%208.png)

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%209.png](/images/2020-11-27-2/Untitled%209.png)

**Shuffler(OSDI'16)**[[paper](https://www.usenix.org/system/files/conference/osdi16/osdi16-williams-king.pdf)][[slides](https://www.usenix.org/sites/default/files/conference/protected-files/osdi16_slides_williams-king.pdf)]:

Function-level re-randomization

用一个独立的线程进行代码拷贝；函数地址为index，真实地址保存在%gs寄存器指向的内存区域中；栈上的返回地址用xor加密。

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2010.png](/images/2020-11-27-2/Untitled%2010.png)

**ILR(S&P'12)**[[paper](http://www.ieee-security.org/TC/SP2012/papers/4681a571.pdf)]

Instruction-level randomization

静态重写；指令打散；通过Fallthrough Map记录执行顺序；通过process-level virtual machine (PVM)执行

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2011.png](/images/2020-11-27-2/Untitled%2011.png)

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2012.png](/images/2020-11-27-2/Untitled%2012.png)

# Definition

- 作者为gadget进行了分类，Turing-complete(TC) gadget set, priority gadget set, MOV TC gadget set
- Turing-complete(TC) gadget set：包含了访存、赋值、算术、逻辑操作，分支、函数调用、系统调用操作
    - 一定需要找到图灵完备的gadget set才能进行ROP攻击吗？
    - priority gadget set：作者从Metasploit中挑选出的10个最常用的gadgets
    - MOV TC gadget set：x86中的mov指令是图灵完备的。[Proof](https://drwho.virtadpt.net/files/mov.pdf)
    - payload gadget set：三个real-world ROP payload中使用的gadget
- re-randomization的间隔时间上界$\tau$，即攻击者无法在长度为$\tau$的时间内获得Turing-complete gadget set, priority gadget set, MOV TC gadget set, or payload gadget set中的任何一个gadget集合。
- Extended footprint gadgets：gadget中出核心指令外，包含会对其他寄存器/内存状态产生影响的指令
- Minimum footprint gadgets：没有副作用的gadget

# Measurement Methodologies

作者主要针对ROP攻击的三个主要阶段进行了评估，提供了方法论。

### Gadget selection

作者对gadget进行了分类：

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2013.png](/images/2020-11-27-2/Untitled%2013.png)

- Derandomization
    - challenge：如何量化fine-grained代码随机化的影响
    - 解决：统计代码随机化后各种gadget数量
    - 对于单轮fine-grained代码随机化的方案，统计上述各种gadget出现的数量。
    - 对于re-randomization代码随机化的方案，作者选取100个re-randomized后连续的地址空间snapshot进行手动分析。
- System Access

    作者通过统计可用的system gadget和在堆栈、data段中vulnerable library pointers的数量，来量化访问privileged operations的难度。

- Payload Generation

    作者通过测量单个gadget的质量以评估生存gadget chain的质量。作者通过register corruption analysis的方式来评价单个gadget的质量。

    - Register corruption analysis
    - 一个gadget中包含一条核心指令，如下图中的mov eax, edx。如果在核心指令前src寄存器edx被破坏、在核心指令后dst寄存器eax被破坏，那么认为这个gadget是损坏的。

    ![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2014.png](/images/2020-11-27-2/Untitled%2014.png)

# Evaluation

作者从Re-randomization间隔时间上界、指针泄漏位置的影响、对gadget可用性的影响、运行开销的影响、gadget chain质量的影响和获得libc指针的难易程度这六个方面，对应用在多个应用程序、库上的数种随机化方案进行评估。

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2015.png](/images/2020-11-27-2/Untitled%2015.png)

Gadgets set选取：包含11中类型指令的图灵完备gadget set、10个Metasploit中最常用的gadgets、7个MOV TC集合、三个realworld payload中使用的gadgets。

## Re-randomization间隔时间上界

Leak 出各个类型gadgets所需要的最少时间、平均时间：

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2016.png](/images/2020-11-27-2/Untitled%2016.png)

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2017.png](/images/2020-11-27-2/Untitled%2017.png)

11种指令类型图灵完备gadgets set泄露所需要的时间：

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2018.png](/images/2020-11-27-2/Untitled%2018.png)

## 初始泄露指针位置的影响

结论：对泄露出的gadgets完整集合没有影响，对泄露所需的时间有影响

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2019.png](/images/2020-11-27-2/Untitled%2019.png)

## 对gadgets可用性的影响

作者通过分析程序应用fine-grained single-round randomization后各类型gadget数量，来量化随机化方案对gadgets可用性的影响。

指令级随机化减少了80%-90%的MIN-FP、EX-FP。

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2020.png](/images/2020-11-27-2/Untitled%2020.png)

## 对gadget chain质量的影响

通过对每个gadget进行register corruption analysis，来衡量这个gadget的质量。

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2021.png](/images/2020-11-27-2/Untitled%2021.png)

## Libc指针可用性

作者测量了libc指针在栈、堆、data段的分布，发现栈上的libc指针数量远大于堆、data段数量只和，因此作者认为更有必要对栈进行进一步的保护。

![2020-11-16-Methodologies%20for%20Quantifying%20(Re-)rand%20e29e3557aec94d13a9a766ff90d65d2f/Untitled%2022.png](/images/2020-11-27-2/Untitled%2022.png)

# 总结

作者提出了在JIT-ROP威胁模型下对ASLR安全性进行量化评估的方法，并对多款随机化方案应用在多个应用上的安全性进行了评估。

主要测量了各种gadget的数量和质量，以及re-randomization时间上界。

