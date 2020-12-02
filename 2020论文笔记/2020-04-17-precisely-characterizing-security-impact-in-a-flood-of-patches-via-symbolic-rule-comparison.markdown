---
layout: post
title: "Precisely Characterizing Security Impact in a Flood of Patches via Symbolic Rule Comparison"
date: 2020-04-20 19:24:33 +0800
comments: true
categories: 
---

> 作者：Qiushi Wu, Yang He, Stephen McCamant, and Kangjie Lu
> 
> 单位：University of Minnesota, Twin Cities
> 
> 出处：the 2020 Annual Network and Distributed System Security Symposium (NDSS'20)
>
> 原文：[Precisely Characterizing Security Impact in a Flood of Patches via Symbolic Rule Comparison](https://www-users.cs.umn.edu/~kjlu/papers/sid.pdf)

# Abstract

大型软件维护者通常会收到大量的bug报告和patch提交，并需要花费大量精力进行审核确认。一般情况下维护者会根据bug是否造成安全影响对其进行优先级排序，优先级低的bug可能会被推迟修复甚至不修复。如果有漏洞被错误地定为低优先级，就有可能在修复之前被攻击者利用。
造成上述问题的原因是不可靠的人工审计，作者针对这一问题提出了自动化的审计方法，在给定bug的patch的前提下确认bug的安全影响。
作者实现了SID，并在Linux kernel上进行了测试，从54k个有效的commit patch中检测到227个漏洞，准确率为97%。其中有197个之前没有被报告为漏洞，并且有21个还存在于最新的Android kernel中。

<!-- more -->

# Introduction

漏洞是具有安全影响的bug，有很多现有工作尝试从一般bug中自动识别和分类漏洞，但很多只是从文本层面（textual）上进行的区分，例如从bug报告的描述，但是很多情况下bug的报告者都无法对bug的安全影响有充分的了解。

现有的自动化工具也无法充分地支持对bug安全影响的分析。

- 静态分析可以对代码中可能存在的问题进行警告，但是静态分析一般都会做出保守的假设，从而造成大量的误报
- fuzzing或者全程序符号执行可以用于漏洞挖掘，但是会面临状态爆炸和覆盖率的问题

上述方法的主要问题是误报率高和效率低。误报率高需要人工检查确认，效率低则可能无法在攻击者利用漏洞之前进行检测。

作者提出了一种自动化的，可以保证准确率和效率的方法。设定一系列安全规则，假设针对bug的patch都能正确的修复bug，如果：

- 未patch的代码一定会打破安全规则并且
- patch之后的代码一定不会打破同样的安全规则

则可以确定，patch修复了一个bug，并且这个bug具有安全影响。

作者利用程序切片和约束不足（under-constrainted）的符号执行实现了上述方法，精准地对patch涉及的代码进行分析。

作者基于LLVM的静态分析pass实现了上述方法并取名为SID（security impact detection，我猜的）。利用静态数据流分析以及程序切片区分并收集有漏洞的操作和安全的操作；利用符号执行来精确地推断安全影响。
作者从66k个Linux kernel的git commit中提取出54K个合法的commit patch，生成了110K个LLVM IR文件，最后发现227个安全漏洞，涉及到out-of-bound access，UAF，double-free，uninitialized use以及permission bypass，并最终确认了24个新的CVE。

# Background

### General Bugs and Security Bugs

下图的第三行和第六行分别对一个安全漏洞和一个普通的bug进行了检查。缺少第三行会造成一个越界读；缺少第六行只会打印一个无效字符。

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled.png](/images/2020-04-17/Untitled.png)

### Common Security Bugs and Impacts

作者选取并分析了100个在NVD数据库中并且存在有效git commit patch的漏洞，并按照漏洞成因以及涉及的安全影响进行了整理。

作者准备按照bug的安全影响对其进行分析，因为不同的成因可能导致相同的安全影响，并且安全影响要比漏洞成因更加容易分析。

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled%201.png](/images/2020-04-17/Untitled%201.png)

### Patch Model

作者选择从bug的patch确定其安全影响，首先需要识别patch中与安全影响相关的组件，作者主要对其分为三类：

- 安全操作（security operations）：边界检查，权限检查，初始化操作，指针置零等等
- 关键变量（critical variables，CV）：被安全操作处理的值，例如边界检查涉及到的array or buffer index
- 漏洞操作（vulnerability operations）：可能会造成安全影响的操作，包括buffer和array相关的操作，读写相关的操作，指针操作，涉及到关键变量结构体的操作（inode，files）以及资源释放相关的操作

如右图所示，可以认为patch就是在漏洞操作之前添加或者更新的对CV的安全操作

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled%202.png](/images/2020-04-17/Untitled%202.png)

### Problem Scope and Assumptions

作者首先假设所有的patch都正确的修复了bug。

在此基础上，作者选择了四种安全影响以及安全规则：

1. 权限绕过：在进行敏感操作之前进行权限检查
2. 越界访问  ：内存读写需要在对象的有效范围内
3. 未初始化使用：变量必须在初始化之后使用
4. 释放后使用 / 双重释放：对象指针不应该在被释放后使用

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled%203.png](/images/2020-04-17/Untitled%203.png)

# Overview of SID

SID会针对作者定义的安全影响提取patch中的安全操作，关键变量以及漏洞操作等组件，并利用符号执行和约束求解来对安全影响进行确认。下面用一份代码举例说明SID的关键步骤。

### Symbolically analyzing patched code

首先SID会分析patch之后的代码，确认其任何情况都不会打破安全规则。

对于右边代码，SID会首先从patch添加的安全操作中提取关键变量sta_id。利用数据流分析，SID可以识别11，14，16，19，21和23行的漏洞操作。每一对安全操作和漏洞操作对应一组程序切片，SID之后会对每一个slice进行符号执行以收集路径约束信息。

SID会构造三组约束：

- 第一组是从patch的安全操作中收集的约束
- 第二组是从slice的符号执行中收集的约束，用以确认变量是否由于算术运算被修改以及收集路径约束
- 第三组约束表示的是对安全规则的打破，例如对Figure 3中的stations边界之外的内存进行访问。

最后SID对这三条约束进行约束求解，如果是不可解的，就证明patch之后的代码没有打破安全规则。

### Symbolically analyzing unpatched code

如果上一步可以通过，SID会分析未patch的版本，来证明去除patch之后的代码会导致安全规则的打破。两个版本的slice会根据关键变量和漏洞操作进行匹配。

与上一步相同，SID也会构造三组约束。第一组约束会与patch版本取反；第二组与patch版本相同，记录关键变量的算术修改；第三组约束表示的是安全规则的遵循。如果是不可解的，就表示未patch的代码会在patch代码限制条件之外的所有的情况打破安全规则。

### Confirming security impact

如果上述两个步骤最后都是不可解，就可以证明这个patch修复了一个安全漏洞。

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled%204.png](/images/2020-04-17/Untitled%204.png)

## The Workflow of SID

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled%205.png](/images/2020-04-17/Untitled%205.png)

### Pre-processing

预处理阶段会处理git commit信息，确认commit是一个bug-fixing patch。

此外SID会利用前向以及后向的针对patched code的控制依赖分析以及针对patch中涉及到的变量的污点分析来确认patch涉及到的文件。

作者发现大多数情况patch只涉及到单个文件。

收集上述信息后，SID会将两个版本的代码编译到LLVM IR。

### Dissecting patches

这一阶段主要用于识别三个关键组件。

首先识别安全操作，从安全操作中提取关键变量，并针对关键变量进行数据流分析，确认漏洞操作。

此外数据流分析也会收集从安全操作到漏洞操作的程序切片。

得到切片之后，SID还会首先进行符号执行来确认程序切片的路径约束是否可以满足，并丢弃所有不可行切片。

### Symbolic rule comparison

# Design and Implementation of SID

## Preparing Analysis Environment

作者基于LLVM实现了SID，利用多个pass分别进行寻找安全检查，符号执行以及数据流分析。SID由5.6K行C++代码以及1.2K行Python代码组成。

为了避免路径爆炸，作者将循环进行展开，当做if语句来处理。为了处理间接调用，作者使用结构体类型来匹配函数目标。

## Static Analysis for Dissecting Patches

作者对四种安全影响的组件定义如下

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled%206.png](/images/2020-04-17/Untitled%206.png)

### Identifying security operations

- **Bound checks**

    边界检查需要使用条件语句，应该包括比较指令(==, >, <, >=, <=)并且两个操作数均为整数类型

    条件语句的一个分支应该为错误处理（例如返回错误码），另一个分支会继续正常的执行

- **Pointer nullification**

    NULL被赋值给一个指针

- **Initialization**

    对内存用store指令赋值为0，或者调用memset置0

- **Permission checks**

    作者收集了类似afs_permission() and ns_capable()的权限检查函数。

    调用上述函数并且对函数返回值的安全检查。

### Extracting critical variables

除permission bypass之外的关键变量提取都比较容易。
对于permission bypass，SID识别CV需要依赖permisson functions（例如ns_capable()）。此类函数的输入一般包括CV和capability number作为参数。非常量的值（文件对象，inode或者用户的对象）一般是敏感资源，访问这些资源需要权限检查。因此SID会将此类参数识别为CV。

### Slicing to find vulnerable operations

对CV进行前向和后向的数据流分析用来确认漏洞操作。主要对以下漏洞操作进行识别：

- **Out-of-bound access：**首先识别利用CV对数组或者buffer访问的指令当做漏洞操作。此外还会识别常见的将CV当做输入的读写函数（例如memcpy）
- **Use-after-free and double-free：**保守的识别所有的指针解引用操作
- **Uninitialized use：**识别对未初始化的变量的操作。包括指针解引用，函数调用，内存访问以及算术运算操作。
- **Permission bypass：**这类操作涉及的CV的结构体类型一般为kuid_t, inode, file或者相关的指针类型，保守的识别所有对CV的操作。

这一步也会收集到关于CV的涉及到安全操作和漏洞操作的程序切片。

需要注意的是安全操作和漏洞操作之间可能是多对多的映射，每一对映射对应一组程序切片。

一个patch可能涉及多个安全操作和漏洞操作，因此会有多组切片。SID利用函数名或者控制流信息将其patch前后的切片相对应。

SID会首先提取并比较切片的漏洞操作，之后进行控制流的比较，从而确认patched版本和unpatched版本程序切片的对应关系。

### Pruning slices

SID会在以下两种情况下裁剪slice：

- CV是patch新引入的变量，对于未patch的代码不可能用到它们
- 漏洞操作只存在于patch版本，意味着新引入的漏洞操作

### Removing infeasible slices

SID利用符号执行来丢弃不可行的slices。SID使用约束不足的符号执行来收集路径约束。到达slice的最后时SID会尝试对约束进行求解。如果不可解，就会将slice丢弃。这样可以降低误报。

## Symbolic Rule Comparison

### Constructing constraints from security operations

更具体的约束定义。

三个约束分别来自安全操作的约束，每个切片的路径约束以及安全规则的约束。

除边界检查外，其他三种用FLAG表示对应的安全操作是否存在，可以用二进制符号来表示约束，1表示针对CV的安全操作存在。

边界检查的约束为对内存对象的检查值。此外还需要知道CV特定的取值范围，所以用符号化的CV来表示。

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled%207.png](/images/2020-04-17/Untitled%207.png)

### Collecting constraints from slice paths

对CV的算术运算，对CV的安全操作

### Constructing constraints from security rules

对于In-bound access, SID需要利用静态分析来确定buffer的size。

对于栈上的变量，可以较为轻松地获取size的值。

对于堆上的变量，需要后向的数据流分析来寻找allcation site来确定size。如果size是常量，常量值就会被使用。如果是变量，则会使用符号表示。

### Solvability for each slice

利用上述三类约束，SID会利用Z3进行约束求解

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled%208.png](/images/2020-04-17/Untitled%208.png)

# Evaluation

作者分别对SID的准确率，性能和扩展性进行了评估。作者选择Linux kernel作为目标程序，收集了66K个git commit，在预处理之后共得到54651个有效的patch，并生成了110136个LLVM bitcode文件。

作者在64bit Ubuntu 18.04上进行了实验，服务器为32G内存以及6核Intel Xeon W-2133 CPU @ 3.60GHz，但是作者的实验都是在单核上进行的。

## Efficiency and scalability

对于每一个patch的两个版本，SID平均花费0.83秒进行分析。对于所有的测试用例，out-of-bound access占了63%的分析时间，因为其具有更多的程序切片以及需要对更多约束进行求解。

## False Positives of SID

为了精确确认bug是否为真正的安全漏洞，作者手工确认了patch描述以及patch的代码和相关的源码。

对于描述中已经明确说明为漏洞的，作者确认其具有安全影响。对于没有明确说明的，作者人工审计了相关代码并确认是否存在安全影响。

最终作者确认了227个安全漏洞以及8个误报，准确率97%。

作者调查了假阳性案例并总结了原因：

- git commit中存在预防性的patch，即在此patch之前已经进行了安全检查。
- 不精确的静态分析，静态数据流分析收集的程序切片很多是不可行的，但是由于约束不足，一些情况无法被过滤。

![Precisely%20Characterizing%20Security%20Impact%20in%20a%20Floo/Untitled%209.png](/images/2020-04-17/Untitled%209.png)

## False Negatives of SID

作者选取了100个最近的Linux kernel漏洞，并用SID对其patch前后的版本进行检测，最终发现了其中47个漏洞，漏报率53%。作者总结了漏报率高的原因：

- 约束不足的符号执行，即使patched之后的代码不会打破安全规则并且patch之前的代码一定会打破安全规则，保守的符号执行也无法证明它。
- 对安全操作和漏洞操作的覆盖率不足，没有将开发者自定义的函数识别为安全操作或者漏洞操作

# Discussion

### The Under-Constrained Symbolic-Execution Engine

SID的符号执行可以从函数的任意一点开始执行，并且只会执行在dissecting patches阶段静态分析收集到的切片。由于收集到的约束是不完整的，所以是约束不足的，可能造成漏报。（目标实际是不可解，更细化的约束可能使目前的可解变为不可解）

### Avoiding path explosion

SID不是全程序的符号执行，而是只针对静态分析收集的程序切片。此外，大多数安全操作与漏洞操作距离较近，所以大多数切片较短。此外，SID会丢弃超过150个基本块的切片，基于作者统计调查的结果（98.8%的程序切片覆盖范围小于150个基本块）。

### Reducing false negatives

约束不足的符号执行的保守性导致了大量的假阴性，因为作者当前的工具实现中，只从安全操作到漏洞操作之间的路径收集约束。作者表示可以考虑从安全操作开始后向收集约束条件，作者经过测试说这可以降低11%的假阴性。

其次，不完整的安全操作和漏洞操作的集合也导致了假阴性，为了克服这个问题，需要从定制化的函数中收集安全操作和漏洞操作，可能需要用到动态分析，封装函数分析等等。

# 评价

方法比较新，通过patch代码与未patch代码的差分对比来检测bug的安全影响，准确率较高

只针对关键变量的程序切片进行符号执行，大幅度降低运行开销，提高效率（但是也因此造成了大量的假阴性）

目前只能针对四类相对简单的安全影响以及相对简单的patch模式进行判断

只考虑了patch点与漏洞点相距较近的情况，对一些数据流分析中可能遇到的难点也没有进行讨论（可能因为不是重点）

（大家可以看一下原文学习一下写evaluation的方式，作者最主要的实验的结果可能撑不起太多篇幅，但是扩展了各种各样的实验以及情况的讨论，最后evaluation几乎占了篇幅的一半