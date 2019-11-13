---
layout: post
title: "MicroWalk: A Framework for Finding Side Channels in Binaries"
date: 2019-03-28 21:11:12 +0800
comments: true
categories: 
---

作者：Jan Wichelmann, Ahmad Moghimi, Thomas Eisenbarth, and Berk Sunar   

单位：Universität zu Lübeck, Worcester Polytechnic Institute  

链接：https://arxiv.org/pdf/1808.05575.pdf  

会议：ACSAC ’18   

<hr/>

## 1.摘要 
研究背景：攻击者可能通过微架构（cache等）侧信道方式获取密钥，因此能够非人工检测出此类侧信道方式是有意义的。
1. 分析对象：选择software binaries，而不是source code的原因在于 binary运行时存在的信息泄露问题 在source code中观察不到。一些商业密码软件没有公布源码。
2. 分析方向：基于memory和control-flow的side-channel。  
3. 分析方法：基于Dynamic Binary Instrumentation以及Mutual Information Analysis

<!--more-->

## 2.分析环境
1. 硬件环境：DELL XPS 8920，Intel(R) Core i7-7700 processor, 16 GB of RAM。
2. 操作系统：Microsoft Windows 10。
3. 工具：MicroWalk，其架构分为
   - DBI后端：Pin v3.6
   - 二进制分析和信息泄露分析：IDA Pro v6.95
4. 测试对象：Microsoft bcryptprimitives.dll v10.0.17134.1以及Intel IPP v2018.2.185

## 3.MICROWALK分析技术
从三个方面进行安全分析：  
1. 揭示代码实现是否具有信息泄露
2. 定位信息泄露在二进制文件中的位置
3. 衡量秘密与中间状态的依赖性。

### 3.1 信息泄露的分析模型
**前提假设**：
1. 敌手能够访问runtime events，比如内存，执行路径甚至寄存器值。
2. 敌手能够选择及更改系统的任意秘密输入。  


![](/images/2019-03-28/fig3.png)  

### 3.2 获取中间状态
定义leakage vectors：
1. 执行路径：用来检测在输入不同的情况下，程序执行操作是否一致。
2. 内存访问：内存访问模式如果与secret有关，也会泄露信息。  

为了能够检测到这两种类型的信息泄露，需要获取对于所有内存访问以及分支操作的中间状态。  
获取中间状态的步骤：
1. 选定一个secret，生成一系列任意的输入。
2. 执行目标binary，记录以下内容：
   -  memory allocations
   -  branches, calls and returns
   -  memory reads and writes
   -  stack operations.

如使用绝对内存地址，干扰因素：ASLR等。   
利用memory allocations的trace和stack operations来计算相对内存地址。  
这里提到了ARM平台上的乘法操作可能泄露信息，也可以采用类似的方法进行定义分析。  

### 3.3 Preparing State Variables
选择泄露粒度g，丢弃地址的较低log2g比特，计算所有或者部分trace entries的哈希值来获得特定执行状态的有效表示。  
注：这里笔者没有太看懂，有兴趣的读者可以读原文。  

### 3.4 分析信息泄露
使用MI来检测、定位及量化信息的泄露。  
将X是一组均匀分布的输入测试样例，Y是一组可能的内部状态(internal states)，例如执行trace的hashes。  
定义执行的state Ti ⊂ X × Y，其中i指时间点i。经过一系列计算可以得到X与所有出现的状态Yi之间的互信息量。  

### 3.5 对MI分数的解读
whole-trace：分数不为0，则表示代码实现存在信息泄露。不能定位泄露点。  
single instruction：可以定位泄露点。

## 4. MICROWALK架构
设计为流水线，分为三个部分：生成测试用例，tracing以及分析。   
![](/images/2019-03-28/fig4.png)
### 4.1 Investigated Binary
对于setup代码，加载和插桩只做一次。目标函数循环。    
1. 对于libraries：程序中循环调用目标函数，并生成可执行文件。
2. 对于可执行文件：使用in-memory fuzzing技术，在目标函数的始末处注入hook，控制函数的执行。

### 4.2 生成输入
使用伪随机生成器生成随机测试样例。也支持已生成的输入。      

### 4.3 生成trace
定制了pintool，记录的信息有：
- 模块加载情况以及相应的起始和结束地址；
- 已插桩的可执行文件中的虚函数调用情况，以识别测试用例执行的开始和结束;
- 通过堆分配函数（如malloc和free（依赖于平台））分配的内存块的大小和地址，用于解析相对内存地址;
- 堆栈指针修改情况，用于解析相对地址;
- 所有相关模块的branches，calls和returns;
- 被调查模块中内存读写情况

### 4.4 trace预处理
1. 加上共同的trace前缀，包含了setup阶段的分配数据(allocation data)。
2. 计算内存地址的偏移地址。
3. 应用泄露粒度  

### 4.5 信息泄露分析
实现了三种分析方法：
1. 比较trace。对于小的算法来说效果较好。
2. Whole-trace MI。为了效率，通过对信息（如相关内存访问及分支目标等）均编码成64比特整数，使用哈希函数将它们压缩成一个64比特整数。
3. single-instruction MI。对于特定指令i，Ti包括访存地址的哈希值。当访存地址的数量后者顺序变化时，这些哈希值也会发生变化。

### 4.6 手动检查和可视化
程序能够将二级制trace转换为可读的文本表示。开发的IDA插件帮助分析函数的哪些部分泄露了信息。也开发了可视化工具。  

## 5. 案例分析1:INTEL IPP
INTEL IPP是个密码算法库，也是INTEL安全产品的加密后端（例如SGX）。密码库在运行时，会根据平台支持的指令集不同选择最优的实现。    
本案测试处理器支持（Intel AVX2）。  

### 5.1 将MircoWalk的MI分析用于IPP
分为几种测试场景：  
1. 随机的明文/密文来进行加密/解密，随机的消息来进行签名。
2. 随机的对称密钥或非对称的私钥
3. 随机的临时secret。


![](/images/2019-03-28/table1.png)   
结论：  
1. 分析92 million条指令花费了73分钟。
2. 对称加密算法实现（3DES，AES，SM4）比较好，但CTR模式存在一些信息泄露。
3. 所有的非对称加密至少存在一个信息泄露情况。
4. Intel IPP中总共发现了13个泄露情况。

### 5.2 发现Intel IPP的信息泄露
利用可视化工具和IDA Pro对这些泄露进行初步分析。  
![](/images/2019-03-28/table2.png)  
主要根据MI的值，得到的主要结论：  
1. cpModInv会泄露信息。
2. ExpandRijndaelKey会泄露信息。

## 6.案例分析2：MICROSOFT CNG
MICROSOFT没有提供源代码也没有提供文档。  

### 6.1 将MircoWalk的MI分析用于CNG
分析共21 million条指令，花费了31分钟CPU时间，找到了4个不同的信息泄露点。  
![](/images/2019-03-28/table3.png)   

### 6.2 发现CNG中的信息泄露
![](/images/2019-03-28/table4.png)   

结论： 
1. 对于AES，查找表导致信息泄露。
2. 对于RSA，实现常量时间
3. 对于ECDSA和DSA，模逆导致信息泄露。


## 7.相关工作
1. 静态程序分析：基于源代码进行分析，局限性在于不能找到由编译器引入的潜在信息泄露问题，还要一些基于LLVM层次代码进行分析。
2. 符号执行：通过确定 影响运行时行为的符号秘密输入 来量化侧通道泄漏。局限性在于开销大。

以上工作都要求得到源代码。

动态程序分析技术：强调本文工作是第一个在实际的闭源binaries上做的。

## 8.结论
提出了一个可扩展的框架来分析闭源的库，来检测微架构的信息泄露问题。  
Intel修复了一个问题，但微软的没有回复。

### 8.1 Future Work
1. Coverage-based Fuzzing：考虑到非密码算法库的侧信道检测，可以考虑使用Fuzzing技术来生成测试样例。
2. 区分调用图中的信息泄露：在计算互信息时，考虑到调用图。





