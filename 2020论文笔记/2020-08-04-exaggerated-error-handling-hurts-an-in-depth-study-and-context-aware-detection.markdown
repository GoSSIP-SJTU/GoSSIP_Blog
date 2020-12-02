---
layout: post
title: "Exaggerated Error Handling Hurts! An In-Depth Study and Context-Aware Detection"
date: 2020-08-04 01:02:44 +0800
comments: true
categories: 
---

> 作者：Aditya Pakki, Kangjie Lu
> 
> 单位：University of Minnesota
> 
> 出处：CCS 2020
> 
> 资料：[Paper](https://www-users.cs.umn.edu/~kjlu/papers/eecatch.pdf)

## 1. Abstract

操作系统内核里有很多的错误处理代码，本文作者发现如果过度处理错误，反而会影响安全性和可靠性，作者将这种问题命名为EEH (Exaggerated Error Handling) 。EEH可能导致DoS、数据丢失、控制流完整性破坏、内存泄露等问题。作者提出了一种检测EEH的方法：EECATCH。EECATCH以上下文感知的方式来检测EEH。作者实现了EECATCH的原型系统，通过跨过程的、域敏感、上下文敏感的静态分析进行EEH的检测。实验评估后，EECATCH总共报告了104个case，经过人工检查，确认了64个EEH Bug，其中48个已经被确认。

<!-- more -->

## 2. Introduction & Background

如下图所示是EEH Bug的示例及其修复，这个例子里的BUG_ON在传入参数为非0时会让系统崩溃，可以从Patch里看出，其实并不需要让系统崩溃，就返回错误码就足够了。

![Exaggerated%20Error%20Handling%20Hurts!%20An%20In-Depth%20Stud%2065a113fb4b064286847103a68a488e15/Untitled.png](/images/2020-08-04/Untitled.png)

检测EEH Bugs的挑战：

1. 对于相同的Error，EH在不同的Error Context下会不同。举例来说，arch子系统中堆内存分配失败值得内核用崩溃的方式来处理，但是在sound子系统下，只需要输出log就可以了，因此检测EEH必须要上下文感知的方式来检测。
2. 决定Error的严重级别。
3. 识别EH。

本文的贡献：

1. 第一个对EEH类型Bug研究的工作。
2. 上下文感知的方式检测EEH。
3. 开源了EECATCH的原型系统。

## 3. EEH

**Error-handling Primitives。**作者将EH分为三类：Terminating Execution、Logging errors、Ignoring errors。

如下所示是严重级别与EH的对应关系。

![Exaggerated%20Error%20Handling%20Hurts!%20An%20In-Depth%20Stud%2065a113fb4b064286847103a68a488e15/Untitled%201.png](/images/2020-08-04/Untitled%201.png)

**历史EEH bug数据集收集**。作者通过grep linux内核的git commit，找关键词 reduce、remove、replace，以及下表中的Log functions和Terminating functions来定位被修补的EEH Bugs。

![Exaggerated%20Error%20Handling%20Hurts!%20An%20In-Depth%20Stud%2065a113fb4b064286847103a68a488e15/Untitled%202.png](/images/2020-08-04/Untitled%202.png)

如下表所示是数据集中，EEH Bugs的影响统计。

![Exaggerated%20Error%20Handling%20Hurts!%20An%20In-Depth%20Stud%2065a113fb4b064286847103a68a488e15/Untitled%203.png](/images/2020-08-04/Untitled%203.png)

并且作者指出，在不同的上下文里，同一个Error的EH会不一样，作者将其分类为了Spatial Context以及Temporal Context。

Spatial Context：在不同的Linux子系统里，对相同的Error会有不同的EH。

Temporal Context：在不同的阶段，对相同的Error会有不同的EH。

## 4. EECATCH

![Exaggerated%20Error%20Handling%20Hurts!%20An%20In-Depth%20Stud%2065a113fb4b064286847103a68a488e15/Untitled%204.png](/images/2020-08-04/Untitled%204.png)

### 4.1 自动识别Error和EH

通过静态分析和基于模式的检测来识别Error以及EH。

**识别Error。**通过后向的数据流分析来识别Error Variables。

**识别EH。**通过初始化的EH集合（即panic等函数）以及unreachable指令，函数的__noreturn属性，来找到初始化的EH的一些wrapper函数。

**Representation of errors：**比较通用的识别模式就是通过标准错误码例如EINVAL来表示各类错误。

Id**entifying error Variables：**分析出seed，可以避免分析相同的Error code。如下代码所示。

![Exaggerated%20Error%20Handling%20Hurts!%20An%20In-Depth%20Stud%2065a113fb4b064286847103a68a488e15/Untitled%205.png](/images/2020-08-04/Untitled%205.png)

Id**entifying and Classifying Error Handling：**跨过程的前向数据流分析来搜集Error对应的不同EH。

### 4.2 提取二维的错误上下文

Spatial Context包括子系统信息，例如sound和filesystem子系统。

Temporal Context分为三类：Initialization、Finalization、Regular functions。

Initialization Functions：从initcall开始通过前向的控制流分析来得到。以及所有的__init函数。

Finalization Functions：__exit函数，以及它调用的函数。

### 4.3 推断<Error, Contexts>的Severity Level

**推断<Error, Contexts>的Severity Level。**目的是自动化的推断出<Error, Context>应该对应的严重级别。EECATCH通过统计分析的方法来完成这个任务。首先从预先得到的数据集里收集相同的<Error, Context>，然后统计分析这类错误一般是如何处理的，从而推测其对应的严重级别。

### 4.4 通过统计分析检测EEH Bugs

如果实际的严重级别要比推断出的严重级别高，则认为是EEH Bug，这里输出的结果会有Ranking，便于人工分析。

## 5. EECATCH实现

以LLVM Pass的形式实现，使用的LLVM版本为10.0.0。

内核代码编译用O2选项优化，LLVM自带的AliasAnalysis在O0优化选项下进行MayAlias的查询会有比较高的假阳性。

**Terminating Functions。**找所有包含UnreachableInst的函数，以及通过控制流找这些函数的Successors。通过这种方式，作者收集到了62个新的Terminating functions，这些大部分是BUG和panic函数的Wrapper。

**Logging Functions。**在IR上大部分的语义信息都丢失了，被展开成了直接对printk的调用，作者通过Association Mining的方式来收集这些宏。总共找到了112个新的用于Log Error的Error Handlers宏。

## 6. Evaluation

实验环境：64GB RAM and an Intel CPU (Xeon R CPU E5-1660 v4, 3.20GHz) with 8 cores

实验目标系统：Linux Kernel 5.3.0-rc2。

通过allyessconfig生成了18071的LLVM IR bitcode文件。

对于选择不同的Threshold，EECATCH分析的结果如下图所示。

![Exaggerated%20Error%20Handling%20Hurts!%20An%20In-Depth%20Stud%2065a113fb4b064286847103a68a488e15/Untitled%206.png](/images/2020-08-04/Untitled%206.png)

EECATCH总共找到了228万个条件分支，62个新的Terminate Execution，112个新的用于输出Error Log的Error Handlers宏，收集了705个EH函数，13683个init函数，12891个exit函数。

如下表所示是EECATCH最终找到的EEH Bugs。

![Exaggerated%20Error%20Handling%20Hurts!%20An%20In-Depth%20Stud%2065a113fb4b064286847103a68a488e15/Untitled%207.png](/images/2020-08-04/Untitled%207.png)

下表的数据证明了使用上下文感知的方式来检测EEH Bug的重要性，在只有Temporal Context或者Spatial Context或者都没有的情况下，检测效果都不如EECATCH。在只有Temporal Context的情况下，会有假阳性，在只有Spatial Context的情况下，会有假阴性。

![Exaggerated%20Error%20Handling%20Hurts!%20An%20In-Depth%20Stud%2065a113fb4b064286847103a68a488e15/Untitled%208.png](/images/2020-08-04/Untitled%208.png)