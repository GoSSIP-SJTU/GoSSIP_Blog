---
layout: post
title: "Securify: Practical Security Analysis of Smart Contracts"
date: 2018-12-06 16:52:54 +0800
comments: true
categories: 
---



作者：Petar Tsankov， Andrei Dan，Dana Drachsler-Cohen，Arthur Gervais，Florian Bünzli，Martin Vechev

单位：ETH Zurich，Imperial College London

出处：CCS '18

原文：https://dl.acm.org/citation.cfm?id=3243780


## INTRODUCTION

以太坊是仅次于比特币的第二大电子加密货币，其允许非信任的双方在没有可信的第三方情况下进行交易，交易能通过执行一段代码实现（即智能合约）。然而以太坊安全问题频发，开发者编写的智能合约存在很多漏洞，作者针对这个问题开发了Securify这个工具。

<!--more-->

Securify是一个全自动化，并且可扩展的分析智能合约，检测漏洞的工具。Securify基于模式匹配。其能在给定特征的情况下分析智能合约是否存在漏洞。Securify的分析针对bytecode，即智能合约源代码（solidity语言）被编译后的结果。通过对bytecode的分析，得出dependency graph，进而得到合约的语义信息。之后根据给定的特征分析合约语义信息是否满足遵循或违背这些特征，判断合约是否存在漏洞。Securify的输入为智能合约的bytecode或源代码（被编译成bytecode输入工具）以及一系列模式，模式采用domain-specific language（DSL，领域特定语言）描述。输出为具体出现漏洞的位置。用户可以自己书写模式，因此Securify有可扩展性。

Securify的分析流如下所示：

![analysisFlow](/images/2018-12-06/Figure1.png)

Securify已经分析了超过18K的合约，并且对链上实际存在的智能合约进行了分析。

### Main Contributions

主要贡献如下

1. 反汇编，用Datalog描述智能合约

2. compliance 和 violation的模板，能检查给定安全特征是否满足

3. Securify的具体实现

4. 对Securify的实验和评估，显示可用性和正确性

以及，Securify还做了审计的工作，从结果发现Securify更加擅长分析稍大的智能合约。

------

## 1 MOTIVATION EXAMPLES

以下是一个存在漏洞的具体合约，solidity编写。红色标明的部分为漏洞存在的地方。

### Stealing Ether

![StealingEther](/images/2018-12-06/Figure3.png)

由于函数默认的访问限制为public，因此initWallet()函数默认可以被外部调用（向该合约发送交易），导致任何人都可以更改钱包的所有者。

Securify的模式有两大类，compliance & violation

	compliance：满足安全特征，合约安全
	
	violation：违背，不安全

对应上述的合约，Securify可以匹配出violation pattern为，owner的赋值不依赖于交易的发送者。那么对应的安全特征为，owner的写入是必须受限的。

以下是另外一个例子

### Frozen Funds

![FrozenFunds](/images/2018-12-06/Figure4.png)

另一钱包的实现。deposit函数有关键字payable，表明交易发送者可以通过调用该函数来向这个合约转账。这个合约的漏洞在于，它本身没有向其他账户转账的函数，实现的功能是通过调用别的合约作为library实现的。而合约可以被销毁。那么假设有一个攻击者销毁了library，那么调用这个library来实现转账的那个合约里面的资产会被全部冻结。

匹配该漏洞的violation pattern为：

1. 用户可向合约存钱，即stop指令（调用结束指令）不依赖于转账的以太币为0（即可以向合约成功存钱）
2. 对于所有的call指令（调用函数指令），转出的以太币为0（即合约本身没有能力向外转出钱）

## 3 THE SECURIFY SYSTEM

本部分展示整个系统的实现

![system](/images/2018-12-06/Figure5.png)

### INPUTS

1. 合约的bytecode或者solidity源代码（编译成bytecode）
2. patterns，用DSL描述

### STEP1 Decomplie

bytecode在以太坊虚拟机EVM上运行。EVM是一个基于栈的虚拟机。

Securify将基于栈运行的bytecode转换为不基于栈的SSA表达形式，具体做法未详细给出。

之后恢复CFG，其中针对Securify的整体分析方法做了一些优化。

### STEP2 Inferring Semantic Facts

该部分Securify分析合约的 semantic facts，使用Datalog描述。整个过程使用已有的Datalog Solver全自动化工具。分析包含了数据依赖和控制依赖。

首先Securify分析针对指令分析出base fact，比如说

>  l1: a = 4 分析得到
>
>  assign(l1, a, 4)

之后，分析每一个指令后，将得到的所有base facts输入到已有的工具，推断出进一步的语义信息，即semantic facts。

### STEP3 Checking Security Patterns

patterns使用securify自己的语言描述。模式匹配时，Security 迭代指令，处理含有some和all量词的pattern，以及，为了检查推断出的facts, Securify直接向Datalog Solver查询。

Securify的模式匹配局限之处为：安全特征比较宽泛，没有对具体的合约分析

### OUTPUT

模式匹配的结果为

1. 有violation被匹配到，则标出含有漏洞指令的位置

2. 没有任何模式被匹配到（既没有compliance，也没有violation），securify给出warning，指明那些位置可能有漏洞

   >violation有些是几个指令造成的，有些是整体的结果，分别归类为Instruction Patterns 和 Contract Patterns。对于前者，Securify能够定位到具体的指令，对于后者，将整个合约标记为不安全。

Securify减少了人工分析的工作。因为现有的分析工具不能清楚确定检测出的漏洞是否真的是violation，所以还依赖人工分析。而Securify能确定性的给出哪些是warnings（无法判断，需要人工），哪些是明确的violation。

### Limitations

1. 没有处理和数字相关的漏洞，如上下溢出
2. 没有检查指令是否能够被执行，直接假设都能被执行到
3. 假设漏洞都能被exploit，这个能通过写DSL弥补

## 7 EVALUATION

作者主要进行了一下几个实验

1. 对真实的合约做分析，评估Securify的有效性

2. 人工分析上一实验的结果

3. 将Securify与Oyente和Mythril（两个已有智能合约分析工具）比较

   - 注意Oyente和Mythril均是基于符号执行的，因此被这两个工具分析出的结果是必定会被执行到的

4. 评估Securify对memory和storage的分析结果

5. 取得Securify分析时的时间和内存消耗

作者使用的数据集有两个

1. 约25K的bytecode数据集（EVM dataset）
2. 100个solidity数据集（solidity dataset）

实验结果如下：

![EVMdataset](/images/2018-12-06/Figure11.png)

![SolidityDataset](/images/2018-12-06/Figure12.png)

![compare&offset](/images/2018-12-06/Figure13&14.png)

可以看到Securify比已有的两个工具效果好很多，然而Securify的基本方法和这两个工具不同，比较不是很令人信服。
