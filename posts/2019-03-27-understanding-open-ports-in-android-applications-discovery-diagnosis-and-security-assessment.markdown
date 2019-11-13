---
layout: post
title: "Understanding Open Ports in Android Applications: Discovery, Diagnosis, and Security Assessment"
date: 2019-03-27 19:24:55 +0800
comments: true
categories: 
---

作者：Daoyuan Wu, Debin Gao, Rocky K. C. Chang, En He, Eric K. T. Cheng, and Robert H. Deng

单位：Singapore Management University, The Hong Kong Polytechnic University, China Electronic Technology Cyber Security Co., Ltd

出处：NDSS 2019

资料：[原文](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_06B-5_Wu_paper.pdf) 

<hr/>

## Abstract

众所周知，服务器是通过开放TCP / UDP端口来提供服务的，这些开放的网络端口也同样存在于许多Android应用程序中。在本文中，作者是针对这些应用程序提出了第一个开放式分析pipeline，包括 discovery, diagnosis和security assessment，系统地了解Android应用程序中的Open Ports及其潜在威胁。最后发现有15.3％的Android应用程序中开放了网络端口，远高于[之前](https://securitygossip.com/blog/2019/02/18/open-doors-for-bob-and-mallory-open-port-usage-in-android-apps-and-security-implications/)分析的6.8％，其中有61.8％的Open Ports应用程序完全来自于应用中的SDK，Open Ports的应用有20.7％存在使用不安全的API的情况。

<!--more-->

## I. Introduction

根据之前Jia 发表在Euro S&P 17上的[`Open doors for Bob and Mallory: Open port usage in Android apps and security implications`](https://securitygossip.com/blog/2019/02/18/open-doors-for-bob-and-mallory-open-port-usage-in-android-apps-and-security-implications/)，他们使用静态分析工具OPAnalyzer分析了 24,000个应用发现有 6.8%的应用存在Open Ports的行为，其中400个存在安全问题并手工确认了57个。但是作者认为静态分析无法对抗动态加载、高级混淆、复杂控制流等行为，有比较大的局限性。于是作者设计了一个流水线分析Open Ports。

![1551946576208](/images/2019-03-27/assets/1551946576208.png)

首先，作者在Google play上发布了一个众包程序——NetMon，目的是和静态分析相比更有效地收集Open Ports。NetMon被设计用来收集Andorid用户手机上的Open Ports。作者从2016.10.18到2017.7共收集了来自136个国家的3293个设备上2778个app，其中有725个是20个厂商的预装应用。之后这些收集到的数据会上传到在作者部署的服务端Open Ports Analytic Engine上，作者认为这些在详细的技术分析之前，这些来自真实用户的端口数据和app使用情况本身就有比较大的价值，所以会先着重分析这些

 

## II. BACKGROUND AND THREAT MODEL

这里的Open Port特指TCP和UDP端口，BluetoothSocket和NFCSocket以及UNIX domain
socket不在当前讨论范畴，威胁模型如下：

- 攻击者App与目标App安装在同一台设备上，并且没有申请任何敏感权限，只有INTERNET权限以访问网络。
- 远程攻击者和目标App在同一个WiFi或者蜂窝数据网络，并且可以发送 TCP/UDP 数据。
- 攻击者可以诱使受害者打开一个Web页面以加载页面上的恶意Javascript代码 （这种攻击只对仅仅使用固定端口的HTTP端口有效）



## III. DISCOVERY VIA CROWDSOURCING

作者采用发布众包而不是静态分析的方式去分析App开放了哪些端口，是因为众包有如下优势：

1. 可以监测野外的App端口开放情况，包括预装应用
2. 不会有假阳性结果
3. 可以检测到Open Ports以及开放时间
4. 可以覆盖UDP和TCP
5. 比静态分析更高效，不用考虑加固手段

此部分有三个步骤：

1. 手机端端口开放检测阶段
2. 上传服务端Open Ports分析引擎
3. 根据应用安装状况分析出结果

![1552117176954](/images/2019-03-27/assets/1552117176954.png)

### A. On-device Open Port Monitoring

检测端口方式：避免过高的overhead，每5分钟读一次`/proc/net/tcp|tcp6|udp|udp6`，以获取以下信息：

- Socket address
- TCP socket state
- The app UID

![1552117196900](/images/2019-03-27/assets/1552117196900.png)



### B. Server-side Open-Port Analytic Engine

服务端分析引擎是众包所上传数据分析中比较重要的一步，目的是归纳这些端口开放的时间、类型和端口属性以在静态分析时定位代码。

![1552117272189](/images/2019-03-27/assets/1552117272189.png)

#### Step 1: Aggregation

按照协议类型TCP/UDP和监听地址分为12个种类

#### Step 2: Clustering by occurrences

根据12个种类中的同一个分类判断某个端口是固定的端口，还是随机端口。判断规则如下：

> `>80%`则是固定端口
>
> `<50%`则是随机端口
>
> 其他使用启发式判断

#### Step 3: Clustering by heuristics

设定`random range`：32,768-61,000，计算Step1中的每组中的许多未细分的端口，在区间内为Nr，不在区间内为Nf

- Nr > 0 and Nf = 0：random

- Nr > 0 and Nf > 0：random（采取保守策略）

- Nr = 0 and Nf > 0：fixed

### C. Crowdsourcing Results

从2016.10.18到2017.7共收集了来自136个国家3293个用户的3293个设备上2778个app，28%来自美国。共收集了40,129,929个端口监控记录，发现了2,778个Open Ports的应用程序（2,284个开放了TCP、1,092开放了UDP端口）和总共4,954个Open Ports（3,327个TCP端口和1,627个UDP端口）。

![1552118628475](/images/2019-03-27/assets/1552118628475.png)

#### 1) Open Ports in Popular Apps

这部分是对用户自主安装应用的统计

![1552118917291](/images/2019-03-27/assets/1552118917291.png)

#### 2) Open Ports in Built-in App

这部分是对手机设备预装应用的统计，这里去除了在上部分中已经包括的应用，例如手机的预装FB和Amazon

![1552119027401](/images/2019-03-27/assets/1552119027401.png)

值得一提的是，很多开放了UDP 68端口是因为 DHCP服务；开放了5060端口是因为VoIP服务。

还有一些网络发现服务端口，例如1900是UPnP端口，5353是mDNS端口。还有一些厂商特有服务端口.

例如 TCP 7080 和 8230 三星 Accessory Service， TCP 59150 和 59152 端口是LG 的 Smart hare，TCP的 5000 端口和UDP 1024端口是索尼 DLNA technique。



## IV. DIAGNOSIS VIA STATIC ANALYSIS

![1552119529546](/images/2019-03-27/assets/1552119529546.png)

接下来是静态分析的部分，这里的静态分析的目的是借助第三部分所发现的Open Ports情况来发现不安全的API使用和端口开放的原因以及用途。静态分析的工具叫做OPTool，分析的逻辑是先在APK中查找这些端口开放相关的API，然后跟踪这些调用的参数。

### A. Open Port Construction and Our Analysis Objectives

首先收集这些API包括native和java层

Native：socket(), bind(), listen(), and accept()

Java：ServerSocket等11种

![1552119918872](/images/2019-03-27/assets/1552119918872.png)



### B. OPTool’s Design and Implementation

OPTool的设计逻辑是使用`backward slicing graph (BSG)`，然后借助`semantic-aware constant propagation`，分别回溯这些API的端口和地址。实现则是使用dexdump把apk中dex转成plaintext，然后再查找方法。

OPTool是控制流和上下文敏感的，处理了函数调用、数组和静态、实例成员变量的引用、异步执行(e.g., in Thread, AsyncTask, and Handler)、组件间通信和Android生命周期等问题。并且不考虑没有caller的方法的处理。

### C. Static Analysis Experiments

分析了33个Google Play类别中的前9,900个应用程序和来自AndroZoo的1,027个应用程序。作者使用第一组来衡量不同类别的Open Ports应用程序的分布情况。这9,900个应用程序中，有1,061个应用程序及其相应的1,453个TCP Open Ports。在AndroZoo的1,027个应用程序中有459个App开放了端口。

![1552121238382](/images/2019-03-27/assets/1552121238382.png)

### D. Detection of Open-Port SDKs

在这1,520个应用程序中，能够检测到13个Open Ports SDK，这些SDK在数据集中至少影响了三个应用程序。

![1552121447540](/images/2019-03-27/assets/1552121447540.png)

## V. SECURITY ASSESSMENT

A. Vulnerability Analysis of Open Ports

这里作者通过JEB逆向和手工动态分析总结了五中漏洞模型。

![1552121627843](/images/2019-03-27/assets/1552121627843.png)

1. 例如 http://127.0.0.1:1234//filename 这种没有验证的信息泄露
2. 没有经过验证的代码执行
3. 代码鲁棒性不佳导致crash
4. 例如视频缓存或其他大型文件读取导致用户流量偷跑
5. 不安全的数据接口

![1552121903898](/images/2019-03-27/assets/1552121903898.png)



### B. Denial-of-Service Attack Evaluation

这部分作者对一些App开放的端口发起DoS，判断对用户正常使用App的影响。横轴为时间，纵轴为流量大小（单位是packet的数量。）

![1552121976240](/images/2019-03-27/assets/1552121976240.png)