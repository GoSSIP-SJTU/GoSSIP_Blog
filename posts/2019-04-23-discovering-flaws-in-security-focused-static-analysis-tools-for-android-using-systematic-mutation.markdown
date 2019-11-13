---
layout: post
title: "Discovering Flaws in Security-Focused Static Analysis Tools for Android using Systematic Mutation"
date: 2019-04-23 13:26:37 +0800
comments: true
categories: 
---
作者：Richard Bonett, Kaushal Kafle, Kevin Moran, Adwait Nadkarni, and Denys Poshyvanyk

单位：The College of William & Mary

出处：USENIX Security 18

资料：[PDF](http://www.cs.wm.edu/~rfbonett/pubs/usenix18.pdf), [Slides](https://www.usenix.org/sites/default/files/conference/protected-files/security18_slides_kafle.pdf), [Video](https://youtu.be/pmvGazllK2M)

<hr/>

## Abstract

现有移动应用程序分析工具，特别是静态分析工具虽然有很高的覆盖率，但是在分析的精度和性能上有较大的损失——这是静态分析工具trade-off的结果。但是这些工具在设计中不够合理的分析和存在的缺陷通常不会在文档中描述，也通常不被研究人员，开发者和用户所知。作者实现了一个Mutation-based soundness evaluation (µSE) 框架，旨在系统地评估现行的Android静态分析工具并发现，记录和修复存在的缺陷，最终发现了FlowDroid的13个分析缺陷，并与工具的开发人员合作成功修复了其中一个缺陷。

<!--more-->

## Introduction
作者的贡献：
- 介绍了mutation-based的健全性评估的新范例
- 作者设计并实现了µSE用于评估静态分析工具
- 通过评估几种广泛使用的Android安全工具来证明μSE的有效性

## Motivation
作者发现FlowDroid v1.5像大多数静态分析工具一样不支持反射，同时也不支持对Android Fragments的分析。FlowDroid v2.0之后开始支持定位fragments，但是检测功能依旧不完善。而1.5版本的FD已经被至少十三个工具所使用并进一步开发，说明这些工具依旧有至少对fragments检测上的缺陷。最后作者报告了这些缺陷并与开发者修复了检测缺陷。作者认为识别这些缺陷不仅有利于使用户意识到这些工具真正的限制，而且也有利于研究。

## Background on Mutation Analysis
变异分析在软件工程领域通常用作测试充分性标准，衡量一组测试用例的有效性。作者以软件工程的变异分析概念为基础，重新定义这种方法，以有效地改进以安全为中心的静态分析工具。

## µSE

![image-20190422205936213](/images/2019-04-23/assets/image-20190422205936213.png)

首先将Android静态分析工具作为输入进行评估（例如，FlowDroid 或MalloDroid）， μSE根据变异方案（Mutation Schema）执行生成安全算子（即安全相关的变异算子 Security Operator）的应用程序，这些变异方案控制应用程序中算子描述的代码转换的位置（即生成的变异App）。 安全算子表示静态分析工具预期会检测到的异常，因此与所检测的工具的功能密切相关。 未被检测出来的变异体代表这个工具存在的缺陷，对其进行分析可以更广泛地发现和了解工具的缺陷。

## Design

![image-20190422211447180](/images/2019-04-23/assets/image-20190422211447180.png)

第1步中，先确定与评估工具的安全目标（例如，隐私泄露检测）相关的安全算子和变异方案，以及分别扩展到Android的某些特有抽象。

在第2步中，使用安全运算符和使用变异引擎（ME）定义的变异方案来改变一个或多个Android应用程序。在此步骤之后，每个app包含一个或多个变异体。为了最大限度地提高效率，会先使用动态执行引擎（EE）过滤掉Android应用程序中的非执行变异体（Dead Code）。在第3步中，使用安全工具来分析变异的应用程序。

### Security Operators
![image-20190422213504853](/images/2019-04-23/assets/image-20190422213504853.png)
安全算子是指被分析的安全工具所要检测或分析的目标行为代码。
为了准确识别工具所生成可以并且要检测的目标安全问题，作者统计了以下来源的安全算子：
1. Research Papers
2. Open source tool documentation
3. Testing toolkits

### Mutation Schemes
变异方案受很多因素的影响：
- 符合并适用于Android特有的抽象（例如生命周期，Intent等）
	- Activity and Fragment lifecycle 
	- Callbacks	
	- Intent messages
	- XML resource files
- 为了提高覆盖率导致过近似可达性
- 限制在被分析工具的安全目标范围内

## Implementation
### Mutation Engine (ME)
ME基于Android的MDROID和变异框架使用Java实现。ME为给定的变异方案，安全算子和目标应用程序源代码导出所有可能注入点的mutant injection profile（MIP）。 MIP通过以下两种类型的分析之一得出：
- 基于文本的解析和在app资源的情况下匹配xml文件;
- 使用基于抽象语法树（AST）的分析来识别代码中的潜在注入点
μSE采用系统的方法将变异体应用于目标应用程序，并注入变异代码。注入过程还使用基于文本或AST的代码转换规则来修改代码或资源文件。

### Execution Engine (EE)
μSE使用EE动态分析目标应用程序，验证注入的变异体是否会被执行。 EE基于之前Android应用程序自动生成输入的工作之上，通过调整CRASH-SCOPE工具来实现。

## Evaluation
### Executing µSE

![image-20190422220359943](/images/2019-04-23/assets/image-20190422220359943.png)

μSE从7个目标应用程序创建了21个突变的APK，向Android应用程序注入7,584个隐私泄露，其中5,558个潜在的非可执行泄漏使用EE过滤掉，只留下2,026个泄漏在变异的应用程序中被确认为可执行路径。 通过过滤掉大量可能不可执行的泄漏（即超过73％），动态过滤显着减少了手动操作。

### FlowDroid Analysis

![image-20190422220623970](/images/2019-04-23/assets/image-20190422220623970.png)

在FlowDroid中一共发现了13种单独的缺陷种类，证明μSE可以有效地用于发现可以解决缺陷的问题。 在最坏的情况下，研究院需要不到一个小时的时间可以分析出未检测到的变异代码。 在最好的情况下，在几分钟内发现了缺陷，证明使用μSE快速找到缺陷所需的手动工作量是最小的。

### Flaw Propagation Study

![image-20190422220955945](/images/2019-04-23/assets/image-20190422220955945.png)

作者使用了两个较新版本的FlowDroid（即v2.5和v2.5.1），以及其他6个工具（即Argus，DroidSafe，IccTA，BlueSeal，HornDroid和DidFail）是否容易受在FlowDroid v2.0中讨论过缺陷的影响。

作者发现最新版本的FlowDroid（即v2.5和v2.5.1）仍然存在对Fragments检测的缺陷。作者说会与开发人员合作解决方案。表3中可以发现，直接基于FlowDroid的工具（即IccTA，DidFail）与FlowDroid一样存在缺陷。而Argus有些地方借鉴了FlowDroid的设计，所以并没有表现出太多的不足。此外，BlueSeal，HornDroid和DroidSafe也并不容易受到这些缺陷的影响。作者解释说，是因为BlueSeal和DroidSafe虽然类似于FlowDroid使用Soot构建控制流图，并依靠它来识别sources和sinks之间的路径。但是BlueSeal和DroidSafe增强了CFG，因此没有表现出FlowDroid中的缺陷。

# Conclusion

作者借鉴软件工程方向的mutation测试来开发μSE框架，并对Android静态分析工具进行系统安全评估，以发现（未记录的）缺陷，而且还演示了这些漏洞如何传播到继承了相关安全工具的其他工具。