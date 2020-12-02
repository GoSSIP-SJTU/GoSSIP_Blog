---
layout: post
title: "DATA – Differential Address Trace Analysis: Finding Address-based Side-Channels in Binaries"
date: 2019-05-06 14:04:16 +0800
comments: true
categories: 
---
作者：Samuel Weiser, Andreas Zankl, Raphael Spreitzer, Katja Miller, Stefan Mangard, and Georg Sigl   

单位：Graz University of Technology, Fraunhofer AISEC, Technical University of Munich  

出处：27th USENIX Security Symposium (USENIX Security 18)  

链接：https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-weiser.pdf

<hr/>

## 简介

背景：内存访问模式（Memory access pattern）不同，可能导致秘密信息的泄露。在秘密信息不同的情况下，与秘密信息有关的数组读取或分支判断，会导致程序运行时的地址不同，从而导致秘密信息的泄露。因此分析程序，找到程序运行时地址相关的信息泄露很重要。

现状：现有的识别程序信息泄露的工具要么不精确要么覆盖范围不全面。

本文提出了DATA，一种差分地址轨迹分析框架（differential address trace analysis framework），可以检测程序二进制文件中基于地址的侧信道信息泄漏。使用DATA工具全自动地分析OpenSSL和PyCrypto，检测出了未知和已知的泄露。OpenSSL采纳了修改意见。

<!--more-->

## 方法

1. 基于地址的信息泄露可以分为两类：
  
   （1）数据。访问的内存地址取决于秘密输入。
   
   （2）控制流。代码执行取决于秘密输入。
2. 检测方法
  
   （1）静态方法：符号化评估所有程序路径。缺点：false positive高
   
   （2）动态方法：动态分析依赖于精确的程序执行，会引入false negatives

DATA的检测方法：  

![1555919232295](/images/2019-05-06/1555919232295.png)

1. 检测address traces的差异。不同的秘密输入，动态二进制插桩方法。
2. 检测信息泄露。验证第一阶段得到的差异是否真的是与秘密信息有关。
3. 信息泄露分类。评估信息泄露的严重程度。

### 检测地址轨迹差异阶段
t=trace(P(k))：k为秘密输入，P为二进制程序，此时t为程序执行过程中记录的地址轨迹。

定义t=[a0,a1,a2,a3...]来表示一系列执行指令。指令操纵CPU寄存器的时候，a~i~=ip保存当前指令指针ip；指令操作内存时，ai=(ip,d)还包含了内存地址d。

定义diff(t1,t2)算法。对于所有ki,kj，在diff(trace(P(ki))， trace(P(kj))) = 空集的情况下，程序P没有信息泄露。

**STEP1**：首先对两种信息泄露的特征和模型进行定义。

1. 数据泄露特征：同一条指令访问不同的内存地址。

举例来说，Listing1中，假设行数等于代码的地址，key~A~=[10,11,12]，key~B~=[16,17,18]，t~A~=trace(P(key~A~))，t~B~=trace(P(key~B~))。使用三元组(ip, cs, ev)来表示数据泄露。ip：泄露指令地址，cs：导致泄漏的函数的地址列表。ev：一系列泄露数据的地址。

![1555896867486](/images/2019-05-06/1555896867486.png)

diff (t~A~, t~B~) = { ( 17, [0, 20], {11, 01} ), ( 17, [0, 21], {12, 02} ),  ( 17, [0, 22], {13, 03} ) }

2. 控制流泄露：依赖于密钥的分支（key-dependent branch）。Listing 2中，假设k~A~=4=100~b~，k~B~=7=111~b~。

使用三元组（ip, cs, ev)表示控制流泄露。ip：分支点ip。cs：调用栈。ev：两个分支的子轨迹（subtraces）。

![1555899294301](/images/2019-05-06/1555899294301.png)

diff (t~A~, t~B~) = { ( 3, [0], { [ 4, ( 7, R ), ( 8, P ), ( 9, R ) ], [ 5, ( 7, T ), ( 8, P ), ( 9, T ) ] } ) }

**STEP2**：记录地址的轨迹。方法：动态插桩PIN。

**STEP3**：找到轨迹差异。算法的大致思路是，当ip值相同但数据地址(d)不同的时候，即检测到了数据差异（data differences）；当ip值不同时，即检测到了控制流差异（control-flow differences）。

### 检测信息泄露阶段

**STEP1**：对evidence的表示进行统一，称之为evidence traces。

在第一阶段中，可以得到导致轨迹不同的指令（ip）。在此阶段中，执行程序，只收集这些指令相关的轨迹。按照时间顺序，对于数据差异，将数据差异写入轨迹；对于控制流差异，将分支点处的目标地址写入轨迹。

使用多种输入执行目标程序，在二维直方图中累积相同指令的evidence traces。

![1555913733591](/images/2019-05-06/1555913733591.png)

H~full~缺点：需要大量的轨迹来估计它，时间开销大。

本文使用简化的两个直方图来代替，一个是H~addr~，跟踪每个地址的访问总数，省略时间信息（x轴）；另一个是H~pos~，实际上是记录每个evidence traces的长度。

例如，现有三个evidence traces：

ev~0~ = [r1, r2], ev~1~ = [r3, r3, r2, r3, r1], ev~3~ = [r2, r1, r2]

则对于地址[r1, r2, r3]，H~addr~ = [3, 4, 3]；对于长度1-5，H~pos~=[0, 1, 1, 0, 1]

**STEP2**：Generic Leakage Test

对于固定的秘密输入，evidence traces的两个直方图表示为H~addr~^fix^，H~pos~^fix^；对于随机的输入，表示为H~addr~^rnd^，H~pos~^rnd^。如果这些直方图是可区分的，则对应的地址差异构成真正的信息泄漏。

应用方法：Kuiper’s test。该test基本上确定两个概率分布是否源自相同的基本分布。

### 信息泄露分类阶段

测试秘密输入和信息泄漏证据之间的线性和非线性关系。 找到这些关系需要对输入和证据轨迹进行适当的表示。

**STEP1**：evidence的表示

对于多个随机的秘密输入，收集evidence traces，将evidence traces汇总到evidence matrices中，其中每列代表一个唯一的跟踪（和唯一的秘密输入）。每个trace长度和访问地址不一样，因此考虑使用两个matrices来处理，一个是M~addr~^ev^，另一个是M~pos~^ev^。行对应的是traces中可能的地址，在M~addr~^ev^中存储了访问地址的数量。M~pos~^ev^中存储了每个地址在每条trace中的位置。

对于例子ev~0~ = [r1, r2], ev~1~ = [r3, r3, r2, r3, r1], ev~3~ = [r2, r1, r2]，得到的矩阵为：

![1555916637123](/images/2019-05-06/1555916637123.png)

**STEP2**：信息泄露模型。定义秘密输入的哪些部分用来和evidence矩阵进行比较。

目的有两个，一是限制统计测试的范围；二是隐含地量化了敌手从观察evidences中获得的信息。

本文框架设计中支持各种泄露模型。

**STEP3**：特定的信息泄露测试。使用随机秘密输入，执行目标程序n次。重用了信息泄露检测阶段得到的轨迹，首先推导得到M~addr~^ev^和M~pos~^ev^，其次根据选择的泄露模型对秘密输入进行转换存入矩阵M~L~^in^，将M~L~^in^的每一行与M~addr~^ev^和M~pos~^ev^的每一行进行比较，使用随机依赖系数（Randomized Dependence Coefficient，RDC）来确定关系。

## 评估和结果

使用工具：Pin 版本 3.2-81205， GCC版本6.3.0。

评估对象：PyCrypto 2.6.1 和 OpenSSL 1.1.0f（commit 7477c83e15）

**分析结果**

括号表示在命令行使用该算法。

![1555918034983](/images/2019-05-06/1555918034983.png)

图2显示了DATA工具的三个分析阶段的结果。

对于OpenSSL的对称算法实现，AES-NI以及AES-VP不存在信息泄漏问题。但是，通过OpenSSL命令行工具使用AES-NI时，密钥解析会产生两处数据泄漏。其他经过测试的对称算法实现会产生大量的数据泄漏，因为它们依赖于使用密钥相关的查找表，这使得它们容易受到缓存攻击。泄漏模型测试也证实了这些泄漏。具体的结果总结参看原文附录A。

对于OpenSSL的非对称算法实现，本文作者分别在RSA和DSA中发现了两个常量时间漏洞（常量时间实现代码存在问题）。此外，还重新确认了ECDSA中实现wNAF所带来的地址差异。具体泄露函数的细节参看原文附录A。

对于PyCrypto实现，本文作者发现通过共享库中的不受保护的查找表实现会泄漏部分关键密钥信息，具体泄露函数的细节参看原文附录A。

DATA发现的泄漏不一定构成可利用的漏洞。 泄漏分类阶段有助于对其严重程度进行评级，但是，准确的判断通常需要在改进具体的攻击方面付出巨大努力。 本文认为，除非给出好的反驳论点，否则任何泄密都应该被认为是严重的。

## 总结

本文提出了差异地址轨迹分析（DATA）来检测基于地址的信息泄露。该工具可以分析实际的软件。 这显示了DATA在协助安全分析师识别信息泄漏方面的实际相关性，以及开发人员在正确实施对策的繁琐任务中的实际意义。



