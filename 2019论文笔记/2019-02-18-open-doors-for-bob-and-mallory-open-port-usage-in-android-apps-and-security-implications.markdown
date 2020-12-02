---
layout: post
title: "Open Doors for Bob and Mallory: Open Port Usage in Android Apps and Security Implications"
date: 2019-02-18 14:06:32 +0800
comments: true
categories: 
---

作者：Yunhan Jack Jia, Qi Alfred Chen, Yikai Lin, Chao Kong, Z. Morley Mao

单位：University of Michigan

出处：IEEE European Symposium on S&P

资料：[PDF](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=7961980)， [Github](https://github.com/jiayunhan/OPAnalyzer) 

<hr/>

## Abstract

本文中，作者对移动平台上的开放端口使用及其安全影响进行了较为系统的研究。作者设计并实现了一种静态分析工具OPAnalyzer，可以有效分析Android应用程序中易受攻击的开放端口使用情况。作者使用OPAnalyzer，对具有超过100K Android应用程序的数据集进行了漏洞分析。 在作者随后的分析中，近一半的开放端口使用是不受保护的，可以直接远程利用。从已识别的易受攻击的用法中，发现了410个易受攻击的应用程序共956个潜在威胁。作者手动确认了57个应用程序的漏洞，包括在官方市场上下载量为1000万到5000万的应用程序，以及一些设备的预装app。这些漏洞可被利用远程窃取联系人，照片甚至安全凭证，还可以执行敏感操作，如恶意软件安装和恶意代码执行。

<!--more-->
## Introduction

TL;DR

作者的贡献如下：

- 实现了一种开放端口应用程序检测模式，并开发了OPAnalyzer以系统地检测Android应用程序中的开放端口使用情况并检测潜在的漏洞。为了提高准确性，作者改进API以实现权限映射完整性，解决Java反射以及native代码分析。 
- 能够将99％的开放端口已识别使用情况分类为5个不同的使用系列，并发现一些特定于移动设备的方案。作者发现这些使用路径中有近一半没有保护，远程攻击者可以直接触发泄露敏感信息并执行高权限操作。
- 对易受攻击的开放端口使用情况进行深入分析，并构造漏洞利用来验证漏洞。手动确认了57个应用，其中包含市场上流行的应用以及某些设备型号上的预装应用。

## Threat Model

作者把具有开放TCP或UDP端口的移动应用定义为开放端口应用。涵盖了两种类型的开放端口应用程序：

作者只考虑以下三种类型的攻击场景：

1. **Malware on the same device**

   智能手机用户安装的恶意应用程序或恶意软件可以使用netstat命令或proc文件`/proc/<pid>/net/tcp`查找同一设备上的侦听端口并发送利用流量。

2. **Local network attacker**

   对于NAT或使用私有WiFi网络的情况下，共享同一本地网络的攻击者可以首先使用ARP扫描查找可访问的手机IP地址，然后启动目标端口扫描以发现易受攻击的开放端口。 

3. **Malicious scripts on the web**

   当受害用户访问攻击者控制的网站时，手机浏览器中运行的恶意脚本可以通过发送网络请求到本地来利用设备上易受攻击的开放端口。

## OPAnalyzer Approach

![1546857151142](/images/2019-02-19/assets/1546857151142.png)

1. OPAnalyzer将apk文件作为输入，提取Dalvik字节码和native so
2. 分析来自native代码和Dalvik字节码的EP，用于后续的静态分析
3. 基于Amandroid的入口点构建了组件间数据流图（IDFG）和数据依赖图（DDG），它们既是flow也是context敏感的
4. 执行污点分析以分析socket输入和预先计算的敏感API集之间的依赖关系，并输出路径
5. 约束分析器检查所有依赖于远程输入的path
6. 可达性分析器过滤那些从程序EP开始不可达的路径，并注释每个路径的运行时可达性。

### A. Entry Point Analysis

使用apktool把dex文件反汇编成smali然后翻译成中间语言Pilar，分析其中的ServerSocket or ServerSocketChannel API当做EP。对于native方法中的socket调用则是写了一个native code analyzer去分析。

### B. Native Code Analyzer

![1546859124314](/images/2019-02-19/assets/1546859124314.png)

图3是来自真实应用程序的代码片段，该程序在native中accept数据，并将控制流传递给应用程序层。如第4行所示，native code使用系统广播intent，并且Java层接收器捕获intent并触发ActionReceiver。另一种类型的控制流跳转显示在第8行。应用程序从native so中启动AndroidManifest中定义的服务，并且UploadService开始在后台运行。

作者设计并实现了一个native代码分析器，它根据过程间的污点分析捕获这种控制流跳转。Native Code Analyzer将以so文件作为输入，并对反编译之后的汇编代码执行污点分析。污点源是从开放端口接受的socket，而receiver是那些可以启动控制流跳转到应用层的函数调用，例如system()。实现了一个plug-in of IDAPro written in Python。

### C. Sensitive API Selection

作者为了对哪些API的使用是敏感的并对这些敏感API进行分析，需要定义了一系列敏感API集和与其对应的系统权限。于是作者使用PScout所提供的静态分析方法来查找API调用和权限之间的映射。

此外，某些API不受Android权限保护，它们所能访问的数据广义上来说依旧是敏感的。例如，应用获取设备位置并将其存储在`/data/data/com.xxx.xxx/files`中，如果攻击者可以使用开放端口获取数据，那么则会造成leak。但是实际上不会触发除INTERNET之外的任何权限检查，敏感位置数据会被泄露。

作者对于此类情况手动收集应用程序可以异步获取不需要许可的数据的所有源，包括应用缓存，数据库，Preference等，定义为DATA_LEAK的pseudo权限。并将伪权限以及关联的API对将添加到敏感API集。

### D. Usage Path Analysis

OPAnalyzer对源自远程EP的DDG执行污点分析，将socket输入作为源，将敏感API设置为sinks。 它输出socket输入可以触发的所有path，以及到达sinks的所有约束。

#### Constraints analyzer

![1546861774137](/images/2019-02-19/assets/1546861774137.png)

检查敏感API所依赖的使用路径上的所有条件语句。如图5所示，远程输入分为两个子字符串并放入Map中。敏感API SendSMS是依赖于两个约束条件。第8行中定义的约束C1检查Map是否为空，而第9行中的约束C2检查从远程输入传入的命令是否为“SEND SMS”。两个检查都很容易绕过，任意远程攻击者都可以构造输入字符串以绕过检查并触发通过SMS发送的恶意payload。

如果代码中C1与常量比较或C2与预定义的一组普通API（例如，Map.size()，Set.isEmpty()）进行比较，则OPAnalyzer将约束注释为弱。

#### Reachability analyzer

使用静态和动态方法表征路径的可达性。首先，静态分析对应用程序Activity生命周期进行建模，并过滤无法访问的usage path。动态方法标识那些以应用启动时默认打开的端口的usage path。这些路径注释为高度不安全，因为远程攻击者有一个比较大的时间窗口来利用它们。

因为只要检测到端口已打开，本地的恶意软件就可以监视proc文件并利用易受攻击的usage path。动态分析使用基于Xposed框架的函数hook实现。

#### Java reflection

java 反射的处理也是使用启发式的方法，借助IDFG实现类似FlowDroid类似的功能，对代码中的调用反射的方法进行恢复并添加到IDFG中。作者说可以处理应用数据集86%以上的反射。

#### Vulnerability discovery methodology

攻击者需要两个条件才能发起攻击：

1. 在端口打开的窗口时间
2. 绕过path上的所有检查以执行敏感API

### E. Evaluation 

作者从PlayDrone中获得24,000个应用程序来评估工具，其中包含来自Google Play 24个类别中的前1000个应用。在24,000个应用程序中，其中6.8％（1632）具有开放端口功能，并且OPAnalyzer在应用启动时确定可以访问133个弱路径。作者手动识别了113条容易被利用的易受攻击的路径。误报率为15.1％，FP主要来自包含应用程序运行时属性检查的路径，例如当app处于release版本时，如果路径包含debugMode == True，则不会被触发，但会是作为弱路径被输出。

还在国内应用市场的Wormhole应用程序上运行OPAnalyzer以评估OPAnalyzer的假阴性（FN），不对Qihoo360库进行测试是因为该问题仅影响旧的测试版。对于百度SDK，OPAnalyzer会检测手动发现的所有可利用路径，甚至可以发现报告中未报告的新漏洞，例如窃取WiFi BSSID。对于AMap，OPAnalyzer报告了四个使用路径，而没有一个被识别为弱路径，因为它的value静态分析无法确定。

![1546862373111](/images/2019-02-19//assets/1546862373111.png)

## Usage and Vulnerability

图7显示了可以由远程输入触发的敏感权限分类。

![1546863217922](/images/2019-02-19/assets/1546863217922.png)

## Security Implications

![1546863462438](/images/2019-02-19/assets/1546863462438.png)

如表3所示，OPAnalyzer输出956个弱路径。发现将近一半的总使用路径被认为是“弱”。从这些弱路径中，作者确定了三种漏洞类别：敏感数据泄漏（V1），特权远程执行（V2）和DoS（V3）。

- V1：Sensitive data leakage

  SD卡上的数据会因为读写问题导致隐私泄露

- V2：Privileged remote execution

  发送SMS和修改联系人、通过广播Intent实现其他操作

- V3：Denial of service

  被用作DoS的反射器（DR-DOS）
