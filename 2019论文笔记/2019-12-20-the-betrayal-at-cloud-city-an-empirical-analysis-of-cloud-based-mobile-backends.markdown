---
layout: post
title: "The Betrayal At Cloud City: An Empirical Analysis Of Cloud-Based Mobile Backends"
date: 2019-12-20 02:03:27 -0500
comments: true
categories: 
---
> 作者：Omar Alrawi; Chaoshun Zuo; Ruian Duan; Ranjita Pai Kasturi; Zhiqiang Lin; Brendan Saltaformaggio;
>
> 单位：Georgia Institute of Technology; Ohio State University;
>
> 会议：USENIX Security '19
>
> 资料：[原文](https://www.usenix.org/conference/usenixsecurity19/presentation/alrawi) | [网站](https://mobilebackend.vet/)

---

这篇文章是该实验室研究移动应用后端的一系列工作中的一篇，主要聚焦于服务器漏洞分析。本文提出了SkyWalker框架，来自动分析给定应用APK所涉及的后端服务器。

<!--more-->

移动应用后端容易存在漏洞，原因包括

- 服务器软件栈复杂，相对不可控。
- 应用使用的第三方库可能用到了开发者不知情的后端

本工作相较于前人工作的优势在于

- 没有把关注局限在服务层软件上，对系统中各层次的软件都做了分析
- 为开发者提供了修复的指引
- 开放工具，使得过程可复现，可以由开发者自行使用

SkyWalker所分析的漏洞可分为0-day和N-day，N-day主要是通过收集软件版本信息、查询已知漏洞得到的，0-day则主要包括SQLi、XSS、XXE。

## Motivation Example

作者分析了一个游戏，通过指纹可以得到各个服务器的软件栈。对于云服务器，Google会自动patch一些软件，但前提是使用的版本在支持范围内，所以开发者需要把一些软件升级到指定版本。

对于第三方服务器存在的问题，开发者可以考虑上报或者去参加赏金项目。

手工进行以上分析对一般应用开发者很不现实，所以考虑做自动化。

![Motivation](/images/2019-12-20/1573095643666.png)

## 方法

### 后端定义

服务器软件栈被定义为

- HW 硬件或虚拟宿主（但看到后面发现这个没提）
- OS 操作系统
- SS 软件，包括数据库软件、服务器软件等
- AS 应用，开发者自己的代码
- CS 信道，应用和后端的连接方式。

服务器属主定义为

- 1st party：开发者完全控制
- 3rd party：开发者完全不控制
- hybrid：开发者和第三方（如基础设施提供商）共同控制
- unknown

![backend labels](/images/2019-12-20/1573096546347.png)

可能的措施包括（后两种适用于开发者无法控制基础设施的情况）：

- upgrade
- patch
- block
- report
- migrate

### 统计方法

N-day以CVE为准，按class和instance统计。如果一个软件instance有多个CVE，也只算一个instance，因为被认为修复操作只需要一次。

0-day只针对AS，以endpoint为单位统计。

### 实现

![Overview](/images/2019-12-20/1573105227643.png)

代码分析使用的是之前的一个工作SmartGen。静态分析建立调用图，符号执行解约束，动态分析获取API endpoint。

后端标记。作者根据收集到的云服务提供商（CP）、机场（Colo）、第三方库（top 5k app用libscout提取）对应后端的IP地址列表（ipcat数据集），如果一个backend是从第三方库里提取的，直接标记成3rd；否则，根据是否在CP、Colo列表里，标记成hybrid或者1st。

扫描与识别。作者注重了减轻扫描对目标服务器的影响，对服务器的网段进行划分，每次随机选择一个网段。探知服务器存活，向常见端口发送TCP ping（SYN-RST）；用SYN扫描所有端口；最终对存活的端口进行TCP握手连接。最终收集到的信息可以用于识别OS、SS和CS。关于OS，作者使用了NASL，该脚本会分析多种特征，最终得出一个服务器和某种操作系统的指纹的匹配度。

N-day分析。作者手工验证了所有983个结果，全部都是真阳性。如果在扫描和识别时采用UDP (?)，总共可以找出6500个结果，包括很多假阳性。

0-day分析。以SQLi为例，作者准备了一些带有时延的消息和正常消息，在一周里的不同时间进行时间测量，如果时间差和指定的时延相同则认为存在缺陷。XSS使用反射型在返回内容中插入元素；XXE则在服务器上发起一个Web请求。收集到了655个结果，无假阳性。

在网站上作者汇总了第三方库的分析结果供开发者参考；网站允许开发者在验证自己的身份后，提交自己开发的应用。

## 评估

从Google Play上收集了4980个应用，从其中的4740个中提取到了backend，其他的都分析过程中崩溃了。整体上检出问题最多的是AS，最少的是OS，大概自己写的东西问题是最多的；问题最多的三类应用是娱乐、工具、游戏，虽然这个结果没有可比性。

![vulnerabilities overview](/images/2019-12-20/1573117418117.png)

backend label的标记率达到了73%，hybrid和1st还是比3rd要多一些。

![vulnerabilities categories](/images/2019-12-20/1573118005407.png)

然后作者按每个层次讨论了相应的问题。对于OS，问题可以分为使用了已经不再更新的版本，和有更新但是没有安装。1st中的OS问题显著高于3rd，这也说明了很多开发人员并不重视OS的更新。作者提到OS的更新可能会因引起SS和AS的不兼容，SkyWalker的开发者建议会考虑这样的因素，但没有展开讲。

SS的问题则有很多都是由PHP带来的，比Top 3里另外两类加起来还多。另外，不再更新的服务器版本也占了相当一部分比例。

![AS by Installs](/images/2019-12-20/1573123519094.png)

![AS by Language](/images/2019-12-20/1573123571547.png)

AS的问题中，XSS是最多的，SQLi也不少。但XSS危害性不一定很大。从语言上来说，PHP的应用中出现的问题是最多的，但是这也和PHP的普及度有关系。

CS的问题中，收集了约18k条消息，其中HTTP和HTTPS大约各一半，HTTPS稍多。4740个应用中，446个只用HTTP，还有147个只用HTTPS。约20%的使用HTTPS的服务器存在配置错误或者是用了过期的版本，可能被downgrade。此外还存在OpenSSH Bypass的漏洞。作者观察了3k多个应用的HTTP消息，发现有个人信息、设备信息，甚至有6个应用通过HTTP重置密码。

作者尽可能地把分析的结果报给了有关方面。作者提到，基于蜜罐的观测，互联网上的N-day扫描机非常多；随着PHP 5.x和7.x (早期)支持的停止，N-day的问题可能会越发严重。

## 建议与应对

根据服务商角色的不同，所能采取的行为有一些差别。区别：服务商是否控制操作系统和软件栈。

![建议](/images/2019-12-20/1573126917077.png)

其他的建议包括：尽量将底层管理委托给可靠服务商；专人负责维护；设置应急预案；采取防御措施；

为了让实验过程合理合法，作者设置了特别的UA和反向DNS，能够让被扫描者了解本项目并可以声明退出，有一个应用联系了作者；作者采用的漏洞利用手段都是非侵入性的；作者向有关责任方告知了扫描的结果。
