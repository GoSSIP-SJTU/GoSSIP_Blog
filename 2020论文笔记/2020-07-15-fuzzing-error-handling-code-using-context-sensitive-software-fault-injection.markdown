---
layout: post
title: "Fuzzing Error Handling Code using Context-Sensitive Software Fault Injection"
date: 2020-07-15 10:54:24 +0800
comments: true
categories: 
---

> 作者：Zu-Ming Jiang, Jia-Ju Bai, Kangjie Lu, Shi-Min Hu
> 
> 单位：Tsinghua University, University of Minnesota
> 
> 会议：USENIX Security 2020
> 
> 出处：[Fuzzing Error Handling Code using Context-Sensitive Software Fault Injection](https://www.usenix.org/system/files/sec20fall_jiang_prepub.pdf)

## Abstract & Introduction

SFI (Software Fault Injection) vs SFI (Software-Based Fault Isolation) [details](http://www.cse.psu.edu/~gxt29/papers/sfi-final.pdf)？

异常处理代码中可能存在漏洞，这些漏洞可能造成DOS、信息泄露等安全隐患。而目前的fuzzing方式很难覆盖到这些代码，因为这些异常处理代码只在某些错误发生时被调用（内存、网络、文件）。针对异常处理代码的传统测试方式为software fault injection（SFI），SFI通过向程序中引入异常，来测试程序是否能够正常处理异常。而目前的SFI技术只能进行context-insensitive的fault injection，存在较大的局限性。如下图所示，context-insensitive的SFI会将两个malloc都引入异常，因此无法触发bug。

作者提出了一种基于context-sensitive SFI的fuzzing方式，能够有效的发现异常处理代码中的bug。

作者的贡献在于：

- 作者针对于异常处理代码进行了调研，发现42%的异常处理代码与偶尔发生的异常相关联；而现有的fuzzing工具仅能发现少量与异常处理相关的漏洞。
- 作者提出了一个种基于context-sensitive的SFI fuzzing方式，能够动态的根据异常处理代码的上下文注入错误，以覆盖那些传统fuzzer难以触发的异常处理代码。
- 基于该方法实现了一个fuzzing framework，FIFUZZ，能够高效的测试异常处理代码。FIFUZZ时第一个能够基于函数上下文来测试异常处理代码的fuzzing framework。
- 作者对FIFUZZ在9个well-tested and widely-used的C语言程序上进行测试，发现了50个漏洞。并与现有的fuzzing工具进行对比，发现了需要遗漏的漏洞。

<!-- more -->

## Background

### Error Handling Code

程序在运行时，可能会遇到一些异常，比如内存分配失败、网络连接失败。处理这些异常的代码称为Error Handling Code。

主要分为两类：input-related errors、 occasional errors

### Bug Examples in Error Handling Code

ffmpeg中libav的两个漏洞patch：

input-related errors：

sbr->sample_rate为输入，输入0时会发生除0错误。

![image-20200709154026130](/images/2020-07-15/image-20200709154026130.png)

occasional errors：

av_frame_new_side_data为内存分配函数，UAF出现在内存分配失败的异常处理代码中。

![image-20200709151534868](/images/2020-07-15/image-20200709151534868.png)



### Study of Error Handling Code

作者在九个应用程序（vim, bison, ffmpeg, nasm, catdoc, clamav, cﬂow, gif2png+libpng, and openssl）中手动对异常处理代码处理的异常进行分类：其中42%为处理偶然错误的代码。

![image-20200709153051921](/images/2020-07-15/image-20200709153051921.png)



### Study of CVEs Found by Existing Fuzzing

作者研究了数个fuzzing工具发现的CVE，其中31%的漏洞发生在Error handling code中，而在这些漏洞中只有9%的漏洞发生在处理偶然错误的异常代码中。

因此作者认为针对于偶然错误的error handling code的fuzzing技术有待提升。

![image-20200709153118723](/images/2020-07-15/image-20200709153118723.png)



## Design



### Basic Idea

之前SFI工作的不足之处：仅能针对一类API进行SFI，不能做到context-sensitive

SFI-based fuzzing的基本思想是使用error sequence进行错误注入，从而测试异常处理代码。

error sequence为0-1序列，其中的每一项代表每一个error point

![image-20200709194044432](/images/2020-07-15/image-20200709194044432.png)



### Error Sequence Model

Error point用其在函数中的位置及它的调用上下文标识。

![image-20200709195119447](/images/2020-07-15/image-20200709195119447.png)

![image-20200709195434969](/images/2020-07-15/image-20200709195434969.png)



### Context-Sensitive SFI-based Fuzzing

作者提出的基于上下文敏感的SFI fuzzing技术，主要由以下六个步骤组成：

1. Identify error sites，通过静态分析识别出error sites
2. 运行程序并收集运行时信息，如error site的上下文信息、code coverage
3. 根据收集到的运行时信息，生成error sequences
4. error sequence mutation
5. 运行程序，并根据error sequence在特定上下文注入错误
6. 收集运行时信息，生成新的序列&mutation

![image-20200709195614888](/images/2020-07-15/image-20200709195614888.png)

### Mutation

作者通过覆盖率反馈指导error sequence的变异，以提高其fuzz的覆盖率。



#### 初始error sequence变异

![image-20200714130330801](/images/2020-07-15/image-20200714130330801.png)



#### 子序列的生成及变异

![image-20200714130434044](/images/2020-07-15/image-20200714130434044.png)





## Implementation

FIFUZZ由六个部分组成：Error site提取、Program generator（插桩）、Runtime monitor、Error-sequence generator、Input generator、Bug checkers。

![image-20200709195806019](/images/2020-07-15/image-20200709195806019.png)



### Error site提取

FIFUZZ需要识别出程序中的error site，作者研究发现，大多数error site都跟函数调用返回值检查相关，因此FIFUZZ认为：

- 如果一个函数为外部函数调用（库函数）
- 其返回值为pointer或整数
- 且返回值被if语句使用NULL/zero检查的

则认为该函数调用为error site。

另外，由于程序中可能有部分函数调用的返回值未被正常检查，即没有if语句检查函数返回值。FIFUZZ使用了统计分析的方式，设置了一个threshold $R$，当一个库函数 $(被if语句检查的callsite数量)/(不同上下文中的callsite总数) > R$ 时，会将该函数所有的callsite认为是error site。



### 插桩

![image-20200714142711437](/images/2020-07-15/image-20200714142711437.png)





## Evaluation

环境：八核 Intel i7-3770@3.40G、 16GB内存、Clang 6.0、Ubuntu 18.04

实验目标：

![image-20200714142939375](/images/2020-07-15/image-20200714142939375.png)



### Error-Site Extraction

作者在FIFUZZ中将threshold R设置为0.6，error function和error site的识别率分别为52.3%和18.6%。

![image-20200612012721377](/images/2020-07-15/image-20200612012721377.png)



### Runtime Testing

作者针对每一个程序，使用FIFUZZ with ASan和FIFUZZ without ASan进行了24个小时的测试，重复三次。

作者称，使用ASan来配合FIFUZZ虽然能够检测出不会导致崩溃的漏洞，但是ASan会带来2x的开销。

![image-20200709200116091](/images/2020-07-15/image-20200709200116091.png)

作者通过检查alerts的root cause，发现了50个漏洞：

![image-20200709200130549](/images/2020-07-15/image-20200709200130549.png)



### Security Impact of Found Bugs

![image-20200709200151589](/images/2020-07-15/image-20200709200151589.png)



### Comparison to Context-Insensitive SFI

![image-20200709200228543](/images/2020-07-15/image-20200709200228543.png)



### Comparison to Existing Fuzzing Tools

作者使用AFL、AFLFast、AFLSmart、FairFuzz与FIFUZZ对五个旧版本程序进行测试：

![image-20200709200243718](/images/2020-07-15/image-20200709200243718.png)





## Conclusion

覆盖次数较多的分支中的代码能够得到更多的测试，而难以覆盖的代码中很可能还隐藏着漏洞。作者就选取了正常执行过程中难以执行的代码片段进行测试，虽然其中的漏洞在实际运行过程中难以触发、难以被利用，但也算是为fuzzing补上了一块盲区。

异常处理代码中容易出现漏洞且难以通过目前存在的fuzzing方式发现。为了能够有效的测试异常处理代码并发现其中的漏洞，作者提出了一种基于context-sensitive的SFI fuzzing framework FIFUZZ。FIFUZZ能够在不同上下文中触发异常，从而覆盖到更多的异常处理代码并发现其中的漏洞。作者在9个C程序中进行测试，发现了50个新的漏洞。

## High dimension fuzzing ?

High dimension fuzzing 多种输入方式之间的组织

- 多种输入方式的内容（stdin、files、fault injection）

  - Error handling code：stdin、software fault
  - Devices（HYPER-CUBE，NDSS'20）：多种交互方式（MMIO、Port IO），写入指针or随机数据
  - 研究输入数据中的不同字节的重要性（Neutaint，USENIX'20）

- 多种输入方式的顺序（API、kernel module）

  - library：API传入参数、API调用顺序
    - library中的漏洞可能会需要一段符合特定序列的API函数调用（funcA->funcB->funcC->bug）
  - Kernel module、driver：ioctl...

- 不太恰当的例子：

```c
// Useless mutation example
c1 = open("f1", "r").read(1)
c2 = open("f2", "r").read(1)

t1 = c1[0]
if t1 == 0xFF:						// 
  // ...									// 
t2 = c2[0]
if t2 == 0xFF:
  // ...
```

```c
c1 = open("f1", "r").read(1)

t = c1[0]
if t == 0xFF:						// 0x100
  // ...
```

- API example

```c
// API bug example
funcptr = NULL;
gv_buf[100];

void vul(buf, len) {
  memcpy(gv_buf, buf, len);
}

void export_init() {				// export
  funcptr = vul;
}

void export_func(buf, len) {
  if (buf && funcptr) {
    funcptr(buf, len);
  }
}

```

欢迎有想法的同学与我交流～ [xgxiao@sjtu@edu.cn](xgxiao@sjtu@edu.cn)

