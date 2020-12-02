---
layout: post
title: "Smart Locks: Lessons for Securing Commodity Internet of Things Devices"
date: 2018-11-26 16:08:30 +0800
comments: true
categories: 
---

作者：Grant Ho, Derek Leung, Pratyush Mishra, Ashkan Hosseini, Dawn Song, David Wagner

单位：University of California at Berkeley

出处： Asia CCS 2016

原文：https://www2.eecs.berkeley.edu/Pubs/TechRpts/2016/EECS-2016-11.pdf


##ABSTRACT

本文研究了市场上的五款智能门锁，发现了这些智能门锁在设计、部署、交互模型上存在的漏洞。实现了几种攻击并提供了一些防护方案。



##INTRODUCTION

原有的大部分成果，专注于对加密协议的分析。本文将关注点放在智能门锁通用的网络架构的安全隐患，以及，用户与设备之间的交互模型上。

（针对新兴的交互模型，作者提到了一款门锁，它能够在合法用户靠近门锁时自动开锁。）

为了研究智能门锁的安全问题，本文设计了一个智能门锁的安全模型并分析了可从市场上购买到的五款较为流行的智能门锁。
通过分析，发现了三种类型的攻击方式。本文分析的五款门锁中，每一款门锁至少可被其中一种方式攻击。


作者认为，漏洞的产生是由于现有的系统都采用了不安全的机制来判断用户的潜在行为。

<!--more-->

![](/images/2018-11-26/1.png)

#####DGC: Device-Gateway-Cloud architecture


##BACKGROUND
![](/images/2018-11-26/2.png)


##SECURITY ANALYSIS
实验结果：
![](/images/2018-11-26/3.png)
三列数据对应本文提出的三类攻击。

四类攻击者：

1. physically-present attacker

2. revoked attacker
   获取了admin提供的权限。例如，airbnb的租客，或是家政人员。他们合法的拥有一段时间的开门权限。

3. thief
   窃取了admin与门锁绑定的设备。

4. relay attacker
   两个攻击者，一方靠近admin，一方靠近智能门锁。攻击者能够转发蓝牙数据，均未拥有开锁权限。

五款门锁中，有四款，只能通过用户的手机联网，即采用DGC模型。
在此类型下，第二类攻击者只需要手机处于断网条件，即可很容易地实施攻击。

案例分析一：DANALOCK（DGC）
此门锁允许任何权限等级的用户自由地与门锁进行交互，即便用户没有联网。

手机断网情况下，服务器端无法对门锁同步新的授权状态。利用此状态不一致，即可使用过期权限开门。
同样可以利用这种状态不一致，抹去非法开锁的日志记录。

案例分析二：LOCKITRON
此门锁内置WIFI模块，直接与云端相连。
缺点：网络连接出现问题，用户将被拒之门外。
优点：避免了案例分析一中存在的两种威胁。

auto unlocking
开启此功能，无论用户是否拥有合法权限，只要在BLE有效范围内靠近门锁即可自动开锁。已有的研究表明，在智能汽车领域存在类似的利用。
在存在多个门的情况下，几个门的状态不同步。

touch-to-unlock
一名攻击者，伪装成门锁，与admin交互，并抓取蓝牙数据。另一名攻击者，重放此通信过程的数据，与门锁交互。实现开锁。

##DEFENSES
1. 引入访问控制表，记录授权信息。避免状态不同步的问题。
2. 使门锁直接与云端相连

##SEDURE AND USABLE INTENT COMMUNICATION

unlocking intent protocol
针对非预期开锁的漏洞，本文旨在设计一种，既方便快捷，交互方式又简单自然的协议。

结合了geo-fencing和touch-to-unlock
开锁条件是：

1. 用户位于开锁范围内，且是第二次进入。
2. 用户使用绑定过的手机接触门锁。

作者认为更精准的定位，能够增加这种auto-unlocking的安全性。例如NFC，这种距离限制比BLE更加严格。

TBIC Touch-Based Intent Communication protocol

![](/images/2018-11-26/4.png)

作者引入了BAN  Body-Area-Networking的概念。
TBIC协议的核心思想是，当且仅当智能门锁发出了intent signal，授权绑定过的随身设备，可以利用BAN发送一个开锁指令。


##关于用户隐私和日志

作者认为，开锁日志一方面为用户管理提供了便利，另一方面却会一定程度上泄露用户隐私。
