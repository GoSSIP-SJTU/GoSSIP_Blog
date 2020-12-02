---
layout: post
title: "EviHunter: Identifying Digital Evidence in the Permanent Storage of Android Devices via Static Analysis"
date: 2018-12-21 12:44:28 +0800
comments: true
categories: 
---

作者：Chris Chao-Chun Cheng, Chen Shi, Neil Zhenqiang Gong, Yong Guan

单位：Iowa State University, NIST Center of Excellence in Forensic Science - CSAFE

出处：CCS'18

资料：[Paper](https://arxiv.org/pdf/1808.06137.pdf), [Slides](https://drive.google.com/file/d/1BuVSfVueBKidzo8T1cYBQB9jjR7QJuXH/view?usp=sharing), [GitHub](https://github.com/paradox5566/EviHunter)

<hr/>

##ABSTRACT & INTRODUCTION

由于移动网络的发展，智能手机上的数字证据在犯罪调查中发挥着越来越重要的作用。数字证据可以存在于智能手机的内存和文件系统中。虽然内存取证方面有很大的进展，但在针对文件系统的取证仍然比较困难。大多数关于文件系统取证的现有研究依赖于手动分析或基于关键字的静态扫描。手动分析代价很高，而关键字匹配通常会错过不包含关键字的数据。在本文中，作者开发了一个名为EviHunter的工具，用于自动识别Android设备文件系统中的数字证据。

作者认为数据是由应用程序产生的，应用程序的代码包含有关应用程序可能写入文件系统的数据类型以及所写入数据的文件相关的信息。因此，EviHunter首先通过对大量应用程序的静态分析来预先计算App Evidence Database（AED）。 然后，EviHunter将Android文件系统上的存在的文件与AED进行匹配，以识别可存储证据数据的文件，所以构建AED是EviHunter的重点。

事实上，已经有大量的静态分析工具用于检测Android应用程序中从Sources到Sinks的敏感数据流。这些工具可以检测到App会收集GPS并将其保存到文件系统，但是它们并不关心GPS信息写入了哪个文件。作者认为，这些工具是出于安全和隐私检测的目的而设计的，在隐私泄漏方面写入敏感数据的文件的位置并不重要。所以EviHunter在几个方面扩展了Android现有的静态数据流分析能力。

<!--more-->

作者最后使用EviHunter评估了8690个真实应用程序。最后，作者对60个随机抽样的真实应用程序的结果进行了手动验证。EviHunter在识别可能包含证据的文件时达到了90％的精度和89％的召回率。

##BACKGROUND AND PROBLEM FORMULATION

### Android File System

**目录结构**：对于每个App，都可以写入文件到私有目录`/data/data/<package name>`，对/sdcard/目录的读写则需要申请权限。

![1544356469205](/images/2018-12-21/assets/1544356469205.png)

**File access**：作者把App所写入文件的路径区分为硬编码方法或软编码方法。 硬编码即App指定绝对文件路径（例如，/data/data/-com.facebook.katana/files/a.txt）并读取/写入文件。 在软编码方法中，应用程序使用Android API来定位文件，然后对文件进行操作。 

![1544356786944](/images/2018-12-21/assets/1544356786944.png)

###Problem Definition

在取证过程中，对于每个设备上，文件系统上可能有数千个文件由于用户活动产生的文件。作者提取了使用了大约5年的Nexus 7平板电脑的镜像。该设备已安装了90个应用程序（包括系统和用户应用程序），这些App生成了大约19K个文件。当在取证的时候需要识别设备上可能包含某些类型的证据数据的文件，例如GPS位置，访问过的URL。作者把这个问题叫做证据识别问题。

##EVIHUNTER

###A. Overview

![1544357270099](/images/2018-12-21/assets/1544357270099.png)

####App Evidence Database (AED)

AED即包含应用程序生成的文件以及该文件包含的证据数据类型。 AED有三列：第一列包括应用程序的包名称；第二列包括文件路径；第三列表示相应文件可以包含的证据数据的类型。文件路径可以是静态路径也可以是动态路径。如果文件路径不依赖于生成文件的应用程序的执行环境，则文件路径是静态的，否则它是动态的。

#### Matcher

匹配器将设备上的文件路径与AED中的文件路径进行匹配，以识别设备上可能包含的证据数据的文件。 如果设备的应用程序未包含在AED中，那么使用EviHunter分析应用程序并将结果添加到AED中。 如果设备上的文件路径相同，则它们与AED中的文件路径匹配。 

### B. Building the AED via Static Analysis

为了保证AED有较高的覆盖率，以避免漏掉潜在的证据。作者利用静态分析而不是动态分析来构建AED。

####I 预处理

给定一个应用程序，首先使用Soot将应用程序转换为Jimple代码。其次，使用[IC3](http://siis.cse.psu.edu/ic3/)来构建互连组件通信（ICC）模型。最后使用FlowDroid来构建调用图和入口点。最新版本的FlowDroid集成了[IccTA](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=7194581)可以在构建调用图时整合了ICC模型。 FlowDroid还会提取应用程序的包名称，作者将其用作应用程序的标识符。但是FlowDroid不足以构建AED，因为FlowDroid无法识别写入数据的文件路径。

####II Tag for a Variable

作者为每个变量定义不同的tag，通过从调用图中的入口点开始执行前向数据流分析来传播变量tag。tag应该携带足够的信息来识别数据的类型和文件路径。 作者定义的标记结构包括以下信息：

- 证据类型集（EvSet）：变量可以包含的证据数据的类型。 
- 文件路径（路径）：与变量关联的文件路径。（在设计用于检测Android应用中的敏感数据流的传统静态分析工具中，标签通常仅包括数据类型）。

####III Propagation Rules

传播规则的作用是在分析应用程序中的语句时如何更新tag。 规则适用于应用程序的三地址Jimple代码。作者将语句分为两类：

- 非方法调用语句的规则

![1544358327997](/images/2018-12-21/assets/1544358327997.png)

- 方法调用语句的规则

方法调用语句可以是不使用返回值的方法调用，也可以是将返回值赋给变量的方法调用。

在第一种情况下，作者将实例v1和参数v2，v3，···的标记传播到callee中并分析callee中的数据流。 

在第二种情况下，作者将分析被调用者中的数据流并将返回值的标记分配给变量v0的标记。

![1544358598710](/images/2018-12-21/assets/1544358598710.png)

#### IV 多线程和反射

对于多线程，假设线程按照先前的研究按顺序执行。因此，每当某个线程产生并开始运行时，作者就会找到相应的输入方法并重定向分析。例如，java.lang.Thread中方法start()的实例调用将被重定向到其实际运行方法run()。作者通过重定向方法处理专用的Android线程库android.os.AsyncTask和android.os.Handler。

对于反射，只有在可以静态解析反射方法时才分析反射调用。EviHunter使用解析的声明类名和路径信息来确定实际的方法调用并将反射调用重定向到它。

#### V Data-Flow Summary for System APIs

![1544359725954](/images/2018-12-21/assets/1544359725954.png)

**System APIs to construct file paths**

表3显示了用于获取文件路径的一些示例。

![1544359896067](/images/2018-12-21/assets/1544359896067.png)

表4显示了示例API的数据流摘要。

**System APIs for string operations**

此部分总结了数据流总用于字符串操作API。 示例API包括toString()，valueOf()，<init>(String)等。 此外，表5显示了一些示例API的数据流摘要。

![1544360041984](/images/2018-12-21/assets/1544360041984.png)

#### VI Sources and Sinks

在EviHunter中，Sinks是将数据写入文件系统的系统API，而Source是创建证据数据或创建文件路径的源。作者扩展了证据Source的方法并给出了文件路径的来源，并公开了EviHunter使用的Source和Sinks。

**Sources for evidentiary data (EvSet)**

1. 位置：位置包括由WiFi和/或蜂窝数据确定的GPS定位。作者将处理位置数据的Android API视为Source。从现有工具中获得了39种源位置。
2. 文本输入：文本输入是用户输入的字符串数据。从现有工具获得了两种文本输入源方法。
3. 时间：作者从现有工具中获得了16种时间源方法。
4. 访问过的URL：用户可以使用WebView通过浏览器或非浏览器应用访问URL。作者总结了3种这类源方法。

**Sources for file paths (Path)**

作者通过在初步分析结果中对100个动态文件路径进行了采样。通过手动分析代码，作者发现应用程序用于生成动态文件路径的前3种方式包括intent，timestamp和UUID，它们分别占动态文件路径生成的33％，20％和12％ 。

当一个变量被指定为API randomUUID()的返回值时，将变量的Path初始化为`<UUID>`。

当变量被指定为返回系统时间的系统方法的返回值时，将变量的Path初始化为`<timestamp>`。

##EVALUATION

作者使用PlayDrone收集了**8690**个Google Play应用。在使用EviHunter为这些应用构建AED时。有些应用需要很长时间才能完全分析，因此作者为每个真实应用程序分析设置了**3分钟**的超时时间。一共有了583个应用程序提前停止，占应用程序总数的6.7％。

报告的文件可以包括至少一种类型的证据数据，包括位置，访问的URL，时间和/或文本输入。如果文件路径包含三种模式之一<timestamp>，<UUID>和<intent>，则文件路径将被视为动态文件路径。所有其他路径都被视为静态文件路径。大约35％的动态文件路径被视为静态文件路径。因此，表中显示的少量静态文件路径实际上是动态文件路径。

![1544360587978](/images/2018-12-21/assets/1544360587978.png)

作者随机抽取了EviHunter报告中至少包含有一个证据数据的文件的30个应用程序，并在手机上安装了每个应用程序，尽可能多地点击应用程序的按钮，并在可能时输入文本输入。这些应用程序生成了559个文件。EviHunter分析结果为，其中72个可能包含证据数据。作者发现EviHunter的精确度达到了90％，并且在本文考虑的四种证据数据中平均recall为89％。此外，还随机抽样了30个EviHunter没有报告任何包含四种指定类型的证据数据的应用程序，作者手动验证了它们的分析结果，没有发现任何漏报。
