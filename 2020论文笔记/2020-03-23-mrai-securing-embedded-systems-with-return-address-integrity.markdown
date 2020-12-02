---
layout: post
title: "μRAI Securing Embedded Systems with Return Address Integrity"
date: 2020-03-23 23:40:54 +0800
comments: true
categories: 
---

作者：Naif Saleh Almakhdhub, Abraham A. Clements, Saurabh Bagchi, Mathias Payer

单位：Purdue University and King Saud University, Sandia National Laboratories, Purdue University, EPFL

会议：NDSS 2020

出处：[http://hexhive.epfl.ch/publications/files/20NDSS.pdf](http://hexhive.epfl.ch/publications/files/20NDSS.pdf)

---

## Abstract

嵌入式系统已被用于许多方面，但由于其资源限制以及low level programming的缘故，导致可能存在许多bug。嵌入式系统通常是一个更大型系统的一部分，如果该嵌入式系统被攻破，整个大型系统将受到严重威胁。（e.g. WiFi system）

针对backward edge的控制流劫持（ROP）对于嵌入式系统仍存在很大的威胁，因为目前针对于类ROP攻击的防御措施（DEP、stack canary、ASLR）存在效果不佳或需要额外的硬件（TEE）。因此无法广泛应用于目前的嵌入式系统。

作者提出并实现了μRAI，针对于MCUS，基于编译器的，使用Return Address Integrity(RAI) 防止针对backward edge的控制流劫持的工具。其不需要使用额外的硬件，使得其适用于目前主流的MCUS。μRAI是通过将返回地址从可写内存转移到只读内存，并通过一个never spill的寄存器来控制返回地址。

<!-- more -->

作者的贡献：

- 提出了Return Address Integrity(RAI)，RAI通过保护返回地址来缓解针对backward edge的控制流街劫持。RAI并不需要额外的硬件支持。
- 针对异常处理也进行了保护。
- 实现了μRAI的原型，并使用CoreMark、几个代表性的MCUS应用进行测试，运行时开销为0.1%。



![image-20200207165359322](/images/2020-03-23/image-20200207165359322.png)



## THREAT MODEL

- 假设攻击者具有任意地址读、写能力，目标是劫持控制流
- 不同于隐藏敏感信息，作者假设攻击者知道代码布局
- MCUS上运行的是一个单独静态链接的二进制镜像
- 设备上有MPU，支持DEP，支持两个权限等级

实验使用最常见的32-bit MCUS：ARMv7-M架构



## BACKGROUND

#### Memory layout

ARMv7-M对于所有代码、数据、外设使用单一物理地址。使用MMIO。

![image-20200207163047829](/images/2020-03-23/image-20200207163047829.png)



#### Memory Protection Unit (MPU)

ARMv7-M使用MPU以支持内存区域的强制访问控制，不同于桌面系统中使用的MMU，MPU并不具备虚拟地址映射的功能。



#### Privilege modes

ARMv7-M支持privileged和user mode，异常处理函数（中断、系统调用）通常运行于特权模式。MPU可以设置user mode和privileged mode对于不同内存的访问权限。



#### Core registers

ARMv7-M有16个寄存器：13个通用寄存器R0-R12、SP（Stack Pointer）、LR（Link Register）、PC（Program Counter）。

μRAI使用一个通用寄存器作为其控制返回地址的state register，简称SR。



#### Call instruction types

![image-20200207164846198](/images/2020-03-23/image-20200207164846198.png)

Direct and indirect calls 会将返回地址存入LR寄存器，随后子函数可能会将LR寄存器push到栈上。



## Overview

![image-20200207165027047](/images/2020-03-23/image-20200207165027047.png)





## Design & Implementation

```
Function Keys (FKs): 硬编码的值，调用函数前与SR异或，函数返回后复原

Function IDs (FIDs): FID = SR xor FK，用于标示返回地址。每个FID对应一个返回地址，因此FID是独一无二的。

Function Lookup Table (FLT): FID与返回地址的对应关系表
```

SR寄存器被分割为两个field：SR[Enc]、SR[Rec]，分别用于控制返回地址、控制递归次数。

![image-20200207175113528](/images/2020-03-23/image-20200207175113528.png)

1. SR初始化
2. SR与硬编码的Function Keys异或
3. 不同的FID对应不同的返回位置
4. 当func2返回时，会根据SR的值查找对应的返回地址，直接跳转到改地址
5. 返回后将SR还原

（出现递归的情况：）

6. SR[Rec]++，记录递归的次数
7. 当SR[Rec]>0时，返回`func2_1`；否则根据FLT返回。



> 上图所示的递归为最简单的直接递归，作者称MCUS中资源有限，应尽可能的避免递归。而更复杂的间接递归作者也做了处理，在附录中提到。



何以体现出μRAI安全性？

> 影响μRAI保护后的控制流的两种方式：1. 通过覆盖jump指令，R+X memory 2.操作SR，但SR仅通过XOR于硬编码的值进行操作，也是在code段，R+X



### *SR Segmentation*

上图所示中，μRAI所带来的开销很大程度来源于FLT。如果由于路径爆炸导致FLT内项数过多，FIK也会产生很多重复，造成不必要的开销。

![image-20200209175511853](/images/2020-03-23/image-20200209175511853.png)

优化方案：对SR进行分段，存在调用关系的函数使用SR中的不同段。从而避免路径爆炸所带来的巨额开销。

![image-20200209180126637](/images/2020-03-23/image-20200209180126637.png)

μRAI可以根据待保护的应用程序自动化的分配segments、FKs和FIDs。



### *Call Graph Analysis and Encoding*

μRAI为了计算SR segment的数量和大小、为每个callsite生成FK，将会对call graph进行分析。

FLT<sub>Max</sub>：FLT的最大值，该值为预先设置的。

μRAI通过预设FLTMax，进行多轮DFS来确定SR segment的数量。



然而segment的数量可能会超过SR 32bit的限制，如下图func 10和func 11需要额外的segment。进入func 10、func 11时会通过调用一个syscall，将当前SR存入仅privileged能访问的一块内存（safe region），并用常量重新初始化SR。



![image-20200209180230508](/images/2020-03-23/image-20200209180230508.png)



### *Securing Exception Handlers and The Safe Region*

中断及异常处理句柄则需要在call graph中被视为根节点。保护方式与func10 func11类似，进入时将SR存入safe region并初始化SR，退出时还原。

为了保护safe region，缓解privileged中出现的任意地址写漏洞，μRAI使用type-base CFI保护中断及异常处理函数及其子函数中的每一个store语句。



### *Target Lookup Routine Policy*

如果将SR和FLT中的FID使用一个switch-case结构逐一比较，仍存在一定程度的开销。O(n)

作者使用相对跳转的方式，用空间换时间，将开销降至O(1)

![image-20200209230358214](/images/2020-03-23/image-20200209230358214.png)

（如果SR=[1,1024]，需要将其他项填充为jump ERROR）

FLT的大小等于最大的FID，这样对程序体积带来较大的开销。



### *Instrumentation*

总结一下，μRAI对程序的插桩分为六个部分：

- 在call site之前设置SR为FID，SR(FID)=SR^FK
- 在callsite之后还原SR，SR=SR^FK
- 对函数设置FLT和TLR（FID与返回地址的对应表）
- 修改函数的return为直接跳转至TLR
- 异常处理函数进入、退出时对SR进行保存与恢复
- 为异常处理函数及其子函数添加type-base CFI保护



## Evaluation

作者在5个代表性的MCUS应用（PinLock, FatFs-RAM, FatFs-uSD, LCD-uSD, Animation）以及CoreMark测试集上测试μRAI。

#### *Security Analysis*

作者针对于三种类型的内存破坏漏洞进行了分析：（user mode）

- *Buffer overflow:* 只能够影响栈上的数据，并不能影响返回地址。
- *Arbitrary write:* 攻击者无法修改SR。虽然在特权模式下SR可能会转存到MPU保护的内存区域中，user mode不能访问。另外还使用Software Fault Isolation (SFI) 对该内存进行保护。
- *Stack Pivot:* 同Buffer overflow

#### *Comparison to Backward-edge CFI*

CFI保护下，返回地址为一个set，攻击者仍可以将控制流劫持至set中的某个函数。

![image-20200209230639896](/images/2020-03-23/image-20200209230639896.png)

#### *Runtime Overhead*

平均0.1%。（FatFs_RAM因为修改了代码的布局以及一个hot loop中寄存器的使用）

![image-20200209230445893](/images/2020-03-23/image-20200209230445893.png)

#### *Memory Overhead*

RAM：15.2%

Flash：54.1%

![image-20200209230501488](/images/2020-03-23/image-20200209230501488.png)



## Conclusion

尽管目前提出了许多防御方案，MCUS仍然会受到类ROP的攻击。作者提出并实现了μRAI，通过使用RAI来保护MCUS免受后向控制流劫持。作者在5个现实MCUS应用上测试表明，μRAI的运行时开销为0.1%。并在多个针对返回地址控制流劫持的场景中进行测试，表明其能够缓解此类攻击。