# KUBO: Precise and Scalable Detection of User-triggerable Undefined Behavior Bugs in OS Kernel

> 作者：Changming Liu，Yaohui Chen，Long Lu
> 
> 单位：Northeastern University，Facebook Inc.
> 
> 出处：The Network and Distributed System Security Symposium (NDSS) 2021
> 
> 原文：https://www.ndss-symposium.org/wp-content/uploads/ndss2021_1B-5_24461_paper.pdf
> 

## Abstract

内核中的未定义行为漏洞（Undefined Behavior bugs，UB）会导致严重的安全问题，当前对其进行检测的方法（作者主要对比[KINT](http://www.cs.otago.ac.nz/cosc440/readings/osdi12-final-88.pdf)）受限于内核巨大的代码量，导致可扩展性和精确性不能兼得。作者使用静态分析与符号执行的方法，利用一种名为incremental call chain upwalk的方法降低了分析的复杂度，最终实现的工具KUBO针对Linux kernel检测到23个严重UB漏洞，误报率仅为23.5%。

<!-- more -->

## Introduction

未定义行为漏洞广泛存在于低级编程语言所编写的代码中，包括整数溢出，除零操作，空指针解引用以及越界访问等等。对于OS kernel，可以被触发的UB可能会导致严重的后果，例如从非特权的用户空间完成对内核的利用。

检查UB漏洞的常规思路：针对UB指令进行检查，判断UB条件是否会成立。

先前工作存在的问题主要如下：
1. **为提高扩展性而牺牲精确性。** 内核代码量巨大，常规静态分析方法无法完成分析，因此多数工作选择进行过程内分析。
2. **报告的漏洞无法触发或者触发无害。** 触发漏洞的路径是否可达或者UB条件能否成立，无法进行判断。即使漏洞可以触发，也可能是无害的漏洞。

作者解决方法如下：
1. **只考虑用户空间输入可触发的UB漏洞。**
2. 为了同时达到高可扩展性和精确性，利用**按需，递增的过程间分析**，从每条UB指令开始，后向遍历以检测是否与用户输入相关。后向分析只会针对特定的函数进行分析，特定函数会根据BTI（Bug Triggerability Indicator）计算，所以高效
3. **利用符号执行判断路径是否可达以及UB条件是否成立。** 由于上一条，作者的方法可以避免路径爆炸的问题。

经过测试评估，KUBO用时33小时对2780万行的Linux kernel完成了检测（其中95%的代码在15小时时已检测完成），并发现23个严重漏洞。

此外，作者选取了19个已知UB漏洞进行检测，并成功检测到其中的12个（36.8%）；对新版kernel检测共报告40个漏洞，其中29个经验证为真正漏洞（23.5%）。

## Design and Implementation

作者首先选取了78个CVE，并分为$S_{eval}$（19个）和$S_{survey}$（59个）两组。
![](/images/2021-03-12/upload_9af0e042ad1f16aa8ce214556d5662ca.png)


### Key concepts and terms

#### userspace input

作者表示，$S_{survey}$中有41个漏洞是因为代码未对用户空间输入做充分的验证，并从中总结出三种用户空间输入的方式：
1. **syscall/ioctl的参数。**这些事内核或者设备驱动与用户空间交互的接口。
2. **sysctl/sysfs。**特权程序或者用户可以利用其读写内核某些全局变量。
3. **memory data transfer functions.**例如copy_from_user().

#### UB condition and UB instruction

作者根据UBSan插桩的指令来选取UB指令，即可能导致UB漏洞的指令，下表列举了KUBO检测的UB以及触发UB对应的条件。

![](/images/2021-03-12/upload_077f0f322d7d88c8987bc5ad3681c558.png)


### System overview

KUBO的检测流程如下图所示：
![](/images/2021-03-12/upload_957b9fd8d55cd55ffa2fc43aeaf8bdab.png)


KUBO的检测会从每条可能的UB指令开始，并后向分析其是否可能被用户空间输入触发。作者用以下两个条件判断UB是否会触发：
- **R1:**UB条件涉及的变量完全受用户空间输入影响；
- **R2:**触发UB的路径约束和UB条件的约束必须同时满足。

具体来说：
1. KUBO追踪UB条件涉及变量的数据与控制依赖。
2. 如果R1无法在当前函数确定，利用BTI判断是否进行过程间分析，当BTI返回true时，KUBO可以判断变量与用户空间输入相关；否则，KUBO开始对函数进行过程间分析（incremental call chain upwalk，选择特定函数进行分析）。
3. 过程间分析会生成函数的data-flow summary，并重新进行1，2步骤，直到确认R1或者到达深度阈值。
4. 当R1确认后，对所报告的漏洞进行post-bug analysis，以对R2进行判断。

#### Backward userspace input tracking & UB condition analysis

对于UB指令，在函数内对其进行后向切片，将与UB指令有控制和数据以来的指令记录在同一切片。
在此基础上进行正向的路径敏感和成员变量敏感的数据流分析，并利用以下tag对数据进行标记：
- **Userspace input**
- **Const**
- **Not know yet**:当前函数无法确认属性的tag，需要过程间分析向上进行确认。

当数据同时拥有多个tag时，利用N > U > C的规则向左取。
最后如果UB condition variable的tag为U时，表示其被用户空间输入控制；如果为N时，需要利用BTI来判断是否继续后向分析。

![](/images/2021-03-12/upload_2ef4cbeb0c91bf96c7b7eada8501a5f2.png)


#### Bug triggerability indicator(BTI)

![](/images/2021-03-12/upload_21c83731d1a0559ff156a40053a5b758.png)


用来判断一些特殊情况是否bug一定会触发，例如上图sr来自用户空间，即使filp->f_pos拥有N的tag，我们也可以确认bug会触发。

#### Incremental call chain upwalk

作者的后向分析主要考虑解决以下问题：
- 如何避免重复对函数进行分析；
- 如何选择对特定函数进行后向分析以降低开销；
- 如何设置分析深度，在精度和开销之间折衷。

根据dataflow summary，选择向上分析调用当前函数的函数。

##### Per-function dataﬂow summary

dataflow summary是过程内分析，来判断函数参数和函数内的用户空间输入如何传递到函数外（call or return）。
分析是路径敏感和成员变量敏感的，并且是离线生成的。

##### Caller selection

对函数进行分析时，会遍历data summary中调用当前函数的函数，并最终选取会将函数参数和用户空间输入传递到当前函数UB contidion variable的函数向上分析。

##### Number of hops

![](/images/2021-03-12/upload_b7a3afb525749ade3e355f89177aa41c.png)


#### Post-bug analysi

经过上述分析，得到的结果均满足R1，但是会由于不满足R2造成大量误报。作者通过对漏洞触发路径进行符号执行来判断是否可达；再同时对UB condition进行约束求解以判断UB condition是否成立。

此外，作者会进一步判断漏洞是否会触发，方法是对corrupt variable进行进一步探索，寻找对其使用的代码点。

![](/images/2021-03-12/upload_3d70fd0276cb1ddfbffa41a082ee5d91.png)

### Implementation

- **call graph & taint analysis:** 
    - 污点分析复用Dr.Checker的工作，添加了额外的用户空间输入；
    - call graph同样复用前人的工作，通过结构体类型分析，优化了间接调用函数的精度。
- **backward tracking & symbolic solving**
    - 基于LLVM和Z3实现
    - 复用DEADLINE的工作
    - 共16,000行C++代码

## Evaluation

实验环境：
- 10核2.2GHz Intel至强处理器
- 64G内存
- Ubuntu 1604；gcc 5.4.0；LLVM 9.0

最终，作者的检测结果如下图所示，共发现越界读写和拒绝服务攻击在内的23处漏洞。
![](/images/2021-03-12/upload_4576c456641bd526444582897a7122a2.png)
