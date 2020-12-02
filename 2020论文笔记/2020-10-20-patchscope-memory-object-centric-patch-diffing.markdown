---
layout: post
title: "PatchScope: Memory Object Centric Patch Diffing"
date: 2020-10-20 15:33:10 +0800
comments: true
categories: 
---

> 会议：CCS 2020
> 出处：[https://www.cs.ucr.edu/~heng/pubs/PatchScope_ccs20.pdf](https://www.cs.ucr.edu/~heng/pubs/PatchScope_ccs20.pdf)

# ABSTRACT&INTRODUCTION

Patch是重要的软件漏洞修复技术之一。Patch diffing一般指的是patch前后程序的二进制代码差异分析的技术。作者发现目前的patch diffing技术并不能够能够帮助安全研究人员理解patch的差异（粒度过细or过大）。

因此，作者对security patch过的程序进行大规模的研究，并对这些patch的模式进行了归类。为了帮助安全分析人员了解patch的详细信息、定位漏洞的root cause、检出有漏洞的patch，作者设计了一种基于动态分析的patch diffing技术PatchScope。

PatchScope基于两个关键的研究结果：1、程序处理输入的方式包含了大量的语义信息；（需要使用对应的结构体转存输入）2、大多数内存破坏漏洞的patch通过更新输入相关的输入数据相关的数据结构来处理（比如 buffer长度）。

PatchScope通过一种semantics-aware program representation，内存对象访问序列（memory object access sequence）（该序列可以对程序是如何使用数据结构来操作输入的数据进行抽象），来进行patch diffing。

<!-- more -->

作者的贡献包括：

- 从code change的角度对security patch进行大规模的的研究，将其中对代码的修改分为九类。并指出了目前在binary level定位patched code存在的一些挑战。
- 作者提出了一种patch diffing的技术。其根据程序如何使用输入相关的一些数据结构来操作输入的数据来进行。不仅对复杂的patch具有鲁棒性，还可以提供丰富的语义信息，以减轻安全研究人员进行逆向工程的负担。
- 证明PatchScope的有效性。优于现有的patch diffing技术，定位的结果更加简洁、准确。

PS：下文中所有出现的patch均指的是security patch，即针对漏洞的修补补丁

# BACKGROUND & MOTIVATION

## Problem Setting

$P$：存在漏洞的程序

$P'$：P的漏洞被patch后的程序

$P$和$P'$都是stripped的binary

作者的目标是，通过比较$P$和$P'$在同一份POC下的execution trace，

- 定位到patch的位置
- 根据发现的差异，提取出更多的语义信息，以帮助安全研究人员分析该patch

## Security Patch Patterns

作者对近年来发表在安全会议中的五个针对开源代码security patch分析工作中的数据集进行了大规模人工分析。选取了其中与内存破环漏洞相关的patch进行分析，去重后共2205个patch。

### Types of patch patterns

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%201.png](/images/2020-10-23/Untitled%201.png)

- 1、2：针对于input的检查，检测非法输入最直接的手段
- 3、4、5：修改input相关的数据结构，包括改变数据类型(int→unsigned int)、修改buffer的大小、将一个用于接收input的buffer从栈上转移到堆上
- 6、7、8、9：修改函数及参数，比如不安全的libc函数(strlen→strnlen、sprintf→snprintf)，以及与程序逻辑相关的漏洞

作者表示，之前的patch分析技术主要针对No.1、2两类patch，而无法有效的分析其他类型的patch。以Figure 1为例，不像典型的patch直接block掉输入，而是根据输入长度将栈分配改为了堆分配，仅修改了4行代码。这样的patch就是通过修改数据结构来fix漏洞的。

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%202.png](/images/2020-10-23/Untitled%202.png)

### Impact on Code Changes

Security patch patterns对binary的影响存在两个比较极端的方面。

- 仅对control flow graph(CFG) or call graph(CG)有修改的patch，增加新的branch、basic block、functions。在所有patch中占71%。（No.1、7、8、9）
- 对CFG，或AST，或program dependency graph没有影响的patch，比如修改buffer的size、格式化字符串 将"%s"改为"%39s"。

## Limitations of Existing Work

目前的binary diffing方面，主要有四类的研究问题：

- literally identical, 逐字比较相同
- syntactically equivalent, 句法等价（control flow/call graph）
- semantically similar, 语义相似（动态的行为、system call序列）
- slightly modified, 略微修改？

作者接下来分析了四种binary diffing技术的优劣：

### Syntax-based binary diffing

BinDiff, Diaphora, and DarunGrim是工业界常用的binary diffing的工具。这些技术通过一系列关于CFG/CG structures, basic blocks, and instructions的启发式算法来计算相似度。

但是这些技术鲁棒性和准确性存在缺陷，过小的修改（buffer size、修改变量类型、修改函数参数）在汇编代码中体现不出足够的差异；过大的修改（较大幅度的code changes、重写一个函数）会造成这些工具会报出大量的low-level code differences，大量的误报会使得安全研究人员头疼 🤯。

Figure 1中的修改，bindiff会报出约30处differences。

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%203.png](/images/2020-10-23/Untitled%203.png)

作者认为bindiff存在两个方面的问题：

- 对齐不准确，左19应该对应的是右4
    - PatchScope使用动态的trace，比静态分析更具有鲁棒性
- low-level differences 并不能够帮助安全研究人员理解这个漏洞（如何触发）
    - PatchScope不仅能够定位出这个bug，还可以更方便的让安全研究人员发现，通过将栈上buffer移到堆上引入了一个新的攻击面。

### Symbolic execution for binary diffing

有的工作通过符号执行来进行binary diffing；或是先通过静态分析找到差异处，再通过符号执行来研究不同的程序行为/影响。

该方案存在的问题：

- 前文提到的，定位patch-relevant changes的鲁棒性、准确性不足。
- 符号执行的输出，通常是一堆指令的符号表达式，人类难以通过符号表达式理解patch
- 通常只是对于一个basic block、循环主体进行符号执行，not scalable（对于patch了函数的，No.6 7 8 9）

### Semantics-aware binary diffing

- 使用system call、库函数API调用来表示程序的语义，常用于clone detection、malware variant comparison。
    - 缺点：粒度过大
- 将一段二进制代码片段视为黑盒，进行动态执行，利用运行时的memory access进行相似度评价。
    - 缺点：粒度过细

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%204.png](/images/2020-10-23/Untitled%204.png)

### AI-powered binary diffing

先将binary code片段转换为vector，再利用机器学习算法计算相似度。

缺点：与Semantics-aware binary diffing类似，粒度太大，无法感知小的修改。

## Design Principles

结合以上的limitation，作者提出PatchScope的设计目标。

- 对于各种类型的patch，在定位patch difference时应当具有鲁棒性。（过小、过大的代码修改）
- 应当输出较为详细的patch difference相关信息。（只有syscall差异看不出啥）
- 应当包含尽可能多的语义信息、high-level的程序表示，以方便安全研究者理解该patch和该漏洞。

# MEMORY OBJECT ACCESS SEQUENCE

## Key Observations

1. 通过各种数据结构操作输入数据包含了大量的语义信息。
    - 在binary-level，编译器通过分配各种内存对象来表示high-level的数据结构，然后使用这些数据结构来操作输入（memory object access）。
2. 大多数patch以添加或修改数据结构的方式来修补漏洞。
    - 表1中大部分类型的patch都会造成不同的memory object access。（input sanitization checks add or update path conditions 会对memory object引入新的逻辑运算）

## Semantics-aware Program Representation

作者使用Memory Object和Memory Object Access来对程序进行抽象。

### Definition 1: Memory Object

记为：$mobj = (alloc,
size, type)$

- alloc：分配时的上下文及分配的位置
- size
- type：栈/堆/寄存器

### Definition 2: Memory Object Access

记为：$A(mobj) = (mobj, cc, op, optype, α)$

- mobj
- cc：所处的上下文
- op：对mobj的操作，考虑三种：data movement instructions, arithmetic instructions, and calling instructions。使用instruction的地址来表示op
- optype：read/write
- α：与mobj相关的连续字节

## MOAS Comparison

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%205.png](/images/2020-10-23/Untitled%205.png)

taint表示该值是否受输入数据影响。

# PATCHSCOPE SYSTEM DESIGN

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%206.png](/images/2020-10-23/Untitled%206.png)

### Dynamic Taint and Execution Monitoring

基于DECAF，一款基于QEMU的全系统二进制代码动态分析框架。

使用了Multi-tag taint analysis，因为需要分析memory object和输入字段之间的关系。PatchScope对输入的每一个字节打上一个unique的tag。

DECAF会记录十分详细的运行时信息，如执行的指令及寄存器的值、可以hook住memory allocation相关的函数（mmap、malloc、alloca）

### Function Call Stack Identification

memory object的分配和访问与它们的上下文是绑定的，因此PatchScope首先要做的是恢复出上下文。最直接的方式是平衡栈帧、call/ret匹配，但是遇到某些编译器优化时可以就会出问题。

常见的编译器优化包括tail call optimization 和 function inline。

- tail call optimization，如下图所示，为了提升性能，编译器会将call B优化为jmp B。
    - 解决：作者使用了近期的一个工作，iCi，其用一系列的启发式规则找出jmp-based inter-procedural calls。
- function inline，对PatchScope而言，由于其是使用memory object access来抽象程序，没有太大的影响，不过会额外报出一些inline function的差异（P inline, P' noinline）。

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%207.png](/images/2020-10-23/Untitled%207.png)

### Excavating Data Structures & Input Fields

因为希望保证合适的粒度（过细：每个字节当一个field；过粗：整个memory object不区分field；最好的是能够像high-level中的数据结构类型一样划分field）。因此PatchScope需要对memory objects进行逆向工程。

basic insight: 通过memory access pattern反映出程序中数据结构的类型。

- Root Pointer Extraction。root pointer指的是数据结构的基地址。static variable和heap分配的比较好处理，local variable的处理比较trick，通过提取第一个出现的指针作为root pointer。
- Memory Object Size Inference。接下来就需要对每个object的大小进行推断，local variables and static variables使用两个连续root pointer的间距作为size。
- Tracking Pointer Propagation。如果没有上下文信息，很难判断一个寄存器中的值究竟是一个地址还是标量。为了经可能的还原high-level中数据结构的field，作者的方案是
    1. 定位出所有的root pointer
    2. 通过root pointer的移动识别出alias pointer
    3. 跟踪对root pointer的算数操作
    4. 最后在load/store时将其转为 $address = base + (index × scale) + offset$ 的形式。
- Correlating Input Fields。在带污点的输入与使用的memory object之间构造关联，作者使用了multi-tag taint传播策略，用启发式的方法，将连续的tainted 访问划分为对同一个memory object的访问。（避免粒度过细）

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%208.png](/images/2020-10-23/Untitled%208.png)

### MOAS Construction

按照Memory Object Access的定义，$A(mobj) = (mobj, cc, op, optype, α)$，构造出memory object access sequence。

### MOAS Alignment

为了识别两个序列之间的差异，需要将两个序列“对齐”。

作者认为最长公共子序列longest common subsequence (LCS)算法着眼于整个序列，并不能得到想要的结果。因此，作者选择了在生物信息学中的Smith-Waterman algorithm，该算法更适合在两个序列中找到局部相同的子序列。

作者将memory object access转化为向量，比较两个向量之间的相似度。

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%209.png](/images/2020-10-23/Untitled%209.png)

# EVALUATION

- 作者在上文提到的五个数据集中选取了37个应用程序，使用同样的编译参数编译出两个binary：unpatched，patched。ground truth通过手工识别patch位置。
- 另外选取了8个source not available的程序，ground truth通过手工调试POC获得。

## 与多个相关工作进行对比

- instructions (BinDiff, Diaphora),
- basic blocks (DarunGrim, DeepBinDiff, and CoP),
- system calls (BinSim),
- memory cells (BLEX),
- memory object access items (PatchScope).

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%2010.png](/images/2020-10-23/Untitled%2010.png)

## 误报率及漏报率统计

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%2011.png](/images/2020-10-23/Untitled%2011.png)

## 能够更好的帮助安全研究人员理解patch

patch中MO、MOA中字段的改变：

![2020-1023-PatchScope%20Memory%20Object%20Centric%20Patch%20D%203840f490ef5a49e6b9e2c2b72dfcebbc/Untitled%2012.png](/images/2020-10-23/Untitled%2012.png)

# DISCUSSION AND CONCLUSION

- PatchScope通过动态污点分析的方式提取程序中的数据结构
- 目前仅支持32位程序
- 开发了一种对内存对象进行动态分析的patch diffing技术，PatchScope。PatchScope不仅能够发现小范围的patch，还可以产生关于该patch更多的细节，以帮助安全研究人员进行分析。

本文对binary diffing的相关工作做了很好的总结和分类，较为详细的说明了目前不同类型binary diffing工作的优略，值得研究binary diffing的同学一读。

作者的工作中也有诸多亮点，比如通过root pointer的传递与运算推断数据结构的各个字段的大小

