---
layout: post
title: "SPIDER: Enabling Fast Patch Propagation in Related Software Repositories"
date: 2020-05-26 15:39:13 +0800
comments: true
categories: 
---

> 作者：Aravind Machiry$^1$, Nilo Redini$^1$, Eric Camellini$^2$, Christopher Kruegel$^1$, Giovanni Vigna$^1$
> 
> 单位：University of California, Santa Barbara$^1$, Politecnico di Milano$^2$
> 
> 出处：41st IEEE Symposium on Security and Privacy (IEEE S&P 2020)
> 
> 原文：[SPIDER: Enabling Fast Patch Propagation in Related Software Repositories](https://machiry.github.io/files/spider.pdf)

## Abstract

开源代码的patch从主分支传播到相关项目会有一定的延迟，对于security patch来说这可能导致安全问题。尽管有类似CVE的漏洞数据库的存在，但是仍然存在两个问题：1）即使是具有CVE编号的漏洞，对其进行修复也有一定的延迟；2）某些security patch没有相关的CVE编号，无法给其他开发人员提供参考。

在本文中作者定义了safe patch，表示在合法输入下不会破坏程序原始功能，无需测试即可接受的patch。作者表示多数security patch可以归到此类别中。作者在此基础上提出了一种通过分析不同版本源码来识别safe patch的技术，并实现了SPIDER。

作者利用SPIDER对32个大型开源项目的341767个patch（包含809个CVE patch）进行了大规模测试，结果显示SPIDER可以识别出其中的67408个safe patch。此外，结果中还包含2278个修复了安全漏洞但是没有CVE编号的patch，其中229个在不同厂商的kernel中还处于unpatched状态，可被认为是未修复的潜在漏洞。

<!-- more -->

## Introduction

相对来说开源代码会比闭源代码更加安全，因为任何人都可能发现代码中的问题并提交patch，但是这也造成了新的问题：

1. 代码维护者需要对大量的bug report以及patch commit进行审核确认，这需要大量人力，并且容易出错。
2. 由于开源代码存在不同分支且不同分支可能经过大量修改，某一分支的patch不能直接移植到其他分支
3. 由于问题2，维护者通常需要经过人工测试以确认patch该如何应用，导致相关漏洞很长时间存在于代码中
4. 由于各种原因，某些security patch没有相关CVE编号，导致其他分支的代码没有修复相关漏洞
5. 攻击者可以扫描开源代码的修复记录，并尝试对相关项目进行攻击。

开源代码的某一分支是否需要接受其他分支的security patch，需要代码维护者理解patch的行为，确认接受patch之后代码不会出现非预期行为。现在有两种主流的方法来简化上述步骤：
* 第一种是利用文本信息，匹配简单的模式或者分析commit message。相关工具虽然因为速度快，轻量级，可扩展等优点被应用于大型代码上，但是文本信息总是不充分的。
* 第二种方法是利用静态分析和符号执行来分析代码语义层面的差别，但是相关技术的开销较大，面临扩展性的问题，无法应用在大型项目上。

一个理想的解决方案是设计一个系统，可以判断patch是否会对程序的正常行为造成影响。如果patch不会影响程序的正常的行为，就可以不经过测试接受patch（作者所谓的safe patch）。为了应用于大型项目，系统应该满足以下两个条件：1）只依赖于原始和patched版本的源码文件，无需其他额外信息(R1)；2）快速，轻量，可扩展(R2)。上述系统可以帮助代码维护者快速确认patch是否为safe patch，从而提高确认和接受patch的效率。

作者的贡献已经在Abstract里提到了，作者的方法致力于确认可以传播到相关项目（且不会造成影响）的patch，并且不需要预先定义特定类型的语义特征（不需要将patch与特定类型的漏洞关联）。此外，作者实现的SPIDER提供了Security Patch mode，可以用于精确识别security patch。

## Safe Patch

safe patch应该满足以下两个条件：
1. **patch之后程序的输入空间不会增加(C1)**  
    patch不会增加合法的输入空间，即patch版本的输入要比原始版本有更多的限制。原始程序接受的输入会造成安全破坏，patch之后的程序将这些视为非法输入。
    $$\forall i \in I|(i\rightarrow f_p )\rightarrow(i\rightarrow f)$$
2. **patch之后，输入相同输出相同(C2)**  
    对于patch版本的程序接受的所有的输入，对应的输出应该与原始版本相等
$$\forall i \in I|(i\rightarrow f_p )\rightarrow(output(i,f_p)=output(i,f)) $$

* $i \rightarrow f$表示输入i可以成功的通过f的执行。即从f的入口基本块开始，给定输入i，控制流不会到达f的任意一个error-handling基本块，这时i是f的有效输入
* 函数的输出包括返回值以及所有函数外部可见的对程序的修改（对heap以及全局变量的写，以及所有函数调用的参数）。对于Listing 1，函数的输出是返回值tlen以及写入指针t->total的值。output(i,f)表示函数f对于输入i的输出值
    
条件1保证了不需要额外增加测试用例。条件2保证了不需要运行现有的测试用例。但是上述条件是充分不必要条件，即满足上述条件的patch是safe patch，但并非所有的safe patch均需满足上述条件

![-w609](/images/2020-05-26/15904266188342.jpg)
以这份代码为例，第3-5行的patch增加了额外的安全检查；第8行考虑了header的长度（最后会减去）。这里的patch是safe patch，因为满足上述两个条件。
![-w590](/images/2020-05-26/15904274744628.jpg)

## Identifying Safe Patches

### Program Dependency Graph

作者识别safe patch的方法用到了PDG。PDG是针对function生成的，$PDG(f)=(V,C,D)$。
其中V表示一系列节点的集合，代表函数中的每一条指令，以及函数入口点$E_n$。
C表示控制依赖图的边，连接存在控制依赖关系的两个节点。方向为src -> dst。标签为T表示当且仅当src为true时dst会执行；标签为F表示当且仅当src为false时dst不执行。如果某一个节点不与其他任何节点存在控制依赖关系，将其与函数入口点相连。
D表示数据依赖图的边，表示src的定义可以到达dst的使用。
![-w1213](/images/2020-05-26/15904285189017.jpg)

#### Control dependency versus control flow

控制流包含所有可能执行的路径，控制依赖只包含到达某一条语句的必要条件

#### Control-Dependency Path and Path Constraint

2(b)中节点3到节点11的路径为（3->10->11）（存在控制依赖关系的两个节点之间的路径只有一条）
路径约束为路径上条件分支的交集

#### Data-Dependency Path

2(b)中节点8到节点15的路径为（8->15）以及（8->11->15）(可能存在多条，与控制依赖相关)

### The SPIDER Approach

SPIDER的输入分别是patch前后版本的函数，检测patch是否为safe patch主要有以下四个步骤：
1. 确认修改的语句
2. 识别error-handling基本块
3. 确认patch之后输入空间没有增加
4. 确认patch之后相同输入对应相同输出

#### Checking modified instructions

首先识别被patch影响的语句，并判断是否可以在满足R1的情况下对函数进行分析。
被patch影响的语句分直接影响和间接影响两种。**直接影响**是指某一条语句的增删改；**间接影响**是指语句与直接影响的语句有控制依赖或者数据依赖。
给定直接影响的语句集合以及函数的PDG，可以确认所有间接影响的语句

**Locally analyzable statement**
局部可分析语句是针对直接影响的语句定义的，表示语句所有的写操作不需要过程间的分析或者指针分析就可以获取。此外，patch对函数的修改不会引入新的函数调用或者指针操作。如果patch之后的函数存在直接影响语句不是局部可分析语句的情况，则此patch不能视为safe patch，因为这种情况需要全程序分析，不满足R1.
![-w579](/images/2020-05-26/15904302662620.jpg)

#### Error-handling basic blocks

识别所有的异常处理基本块($BB_{err}$)，忽略所有对$BB_{err}$内的修改。作者假设patch对$BB_{err}$的修改不会影响程序正常的功能。

#### Non-increasing input space

如果patch没有影响任何控制流语句(if, for, while)，则输入空间不会增加（增大read的size？），否则需要验证原始函数输入空间之外的输入不能成功地执行函数。
首先需要识别合法退出点（valid exit point，VEP），即执行到此语句则表示输入被函数成功地执行了。
合法退出点等于函数所有的return语句减去异常处理到达的return（如果函数是用goto语句进行异常处理，最后到达同一个退出点呢？）
需要验证所有能在$f_p$执行到VEP的输入也可以在$f$中执行到VEP，具体方法是收集$f_p$中执行到VEP的路径约束，确保约束内的输入在$f$中可以执行到VEP。这一步需要用到符号表达式和SAT求解器。

#### Output equivalence

这一步需要验证在相同输入下，函数执行接受后所有外部可见的改变是相同的。
从所有的affected statements中删除控制流语句以及对局部变量的写，保证对非局部变量的写，函数返回值以及调用函数的参数在相同输入下相同。
对于Fig.2，可以确定函数的输出是节点17和节点19。对于节点17，在PDG中存在两条数据依赖路径(8->15->17)和(8->11->15->17)，需要对不同的路径取路径约束，并结合输出的符号表达式，确认patch版本的路径约束是非patch版本对应路径约束的子集，且符号表达式是相同的。
![-w1097](/images/2020-05-26/15904335423730.jpg)


### Implementation

#### preprocessing
作者利用unifdef对输入的源码进行预处理，忽略所有的#include以及宏定义，宏函数会被当作常规函数处理。作者会尽可能地接受所有的#ifdef和#ifndef，尽量让尽可能多的代码存在于预处理之后的代码中。

#### parsing
作者用Joern fuzzy parser将函数转化为抽象语法树以及生成控制流图。作者还在Joern的基础上实现了到达定义分析，简单类型推断以及控制依赖分析。
在这一阶段，SPIDER生成了后续处理需要的AST，CFG以及PDG。

#### fine-grained diff
作者利用函数名在不同版本的函数源码，认为所有对函数的插入，删除，重命名都不是safe patch。
SPIDER使用java-diff-utils来识别patch影响的函数，并利用Gumtree来确认patch版本中被move，insert，delete，update的AST节点。
move表示节点的位置发生变化，但是内容没有改变；update表示位置没有变化，内容被修改。

#### identification of error-handling basic blocks\

![-w443](/images/2020-05-26/15904579074774.jpg)
作者认为满足以下两个条件之一的基本块是异常处理基本块：
1. 如果基本块让函数返回一个常量负值或者C标准库的error code(定义在errno.h，例如EINVAL)，上图的BB2满足此条件
2. 如果基本块的结束是直接跳转（goto）到一个可能是异常处理的标签（panic, error, fatal, err，作者一种定义了15种），上图的BB3满足此条件

#### patch analysis

1. **对循环的处理**  
    作者通过删除PDG中所有的back edge来移除所有的控制依赖和数据依赖循环。back edge指edge从一个节点t主导的节点指向节点t。
    如果patch对循环内的代码有直接影响，则认为patch不是safe patch，因为可能不会满足C2。
2. **符号表达式的转换**  
    对于给定语句，判断其是否有指向它的数据依赖边，递归判断直到找到一个没有incoming edge的节点，从这种节点开始定义符号，从向下依次计算符号表达式
3. **对函数调用的处理**  
    对函数名和函数参数的符号表达式计算hash。例如strlen(buf)可以表示为hash(strlen, sym(buf))
4. **对多定义的处理**  
![-w438](/images/2020-05-26/15904598947094.jpg)
![-w378](/images/2020-05-26/15904599777478.jpg)

Ite表示if-then-else，->表示左边的路径约束更严格。
上图节点7的定义可能来自3和5，其中5的路径约束更严格

#### assumption

SPIDER基于以下几个假设：
1. 假设代码中没有别名依赖，也就是说不需要进行指针分析
2. 函数的输出只与输入有关，即相同的输入一定有相同的输出。函数内函数调用的顺序不会对函数输出产生影响，即f1(arg1);f2(arg2)与f2(arg2);f1(arg1)相等。
3. #if和#else不会对patch的结果造成影响，因为patch代码可能存在于不同的条件下，分析选用哪条分支需要对代码进行精确的分析，对大型项目不具有可扩展性。

### Security Patch Mode

作者在SPIDER中提供了security patch mode(SeP)，用以识别代码中的security patch，并且可以保证**没有误报**。作者表示大多数security patch是对输入进行了额外的合法性检查，所以在SeP中，作者使用了更严格的safe patch定义，认为sp只可以影响控制流语句。

### Evaluation

#### performance

平均3.4s分析一个patch（双核2.4GHz CPU，8GB RAM）

#### Large-scale evaluation

作者选择了32个大型开源项目，收集了从2016年开始的32个月的341767个commits，对SPIDER进行了一系列测试
![-w1008](/images/2020-05-26/15904616908557.jpg)

#### Effectiveness of patch analysis

从341767个commit中识别出67408个safe patch(19.72%)，其中58.72%的safe patch至少在一个fork中没有被接受
（好像论文里没有提到false positive）  

#### Evaluation on CVEs

![-w341](/images/2020-05-26/15904620489163.jpg)

作者表示常规的patch只有19.72%为safe patch，CVE patch有55.37%为safe patch

#### Security patches missing a CVE number

作者使用SPIDER的SeP对所有commit进行了检测，随后检查了检测出的sp是否有CVE编号。SPIDER总共识别了2278个security patch，并手动确认了结果的正确性。

### Limitations

1. **small patch** 作者的方法只能识别行数较少的safe patch，因为作者为了保证sound，加了很多条件和假设。
2. **false negative** 正如第一条所说，作者的实现基于很多前提假设，所以会忽略掉很多不满足作者假设的safe patch。
3. 识别异常处理基本块用到了启发式的规则。
4. 将宏函数当作常规函数处理，恶意的patch可以绕过SPIDER。
5. 工具依赖，理想的检测工具应该足够通用，例如将不同语言的代码转化为通用的表示，再用统一的后端进行处理（LLVM）。作者的实现使用了Joern和GumTree，工具的限制也会对作者的实现产生影响。
