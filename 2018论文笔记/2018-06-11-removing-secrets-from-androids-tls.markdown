---
layout: post
title: "Removing Secrets from Android’s TLS"
date: 2018-06-11 15:38:49 +0800
comments: true
categories: 
---

> Jaeho Lee and Dan S. Wallach, Rice University
>
> NDSS 2018, Session 1B: Attacks and Vulnerabilities

## 摘要

**主要工作** 

+ 对于近期Android发行版中对于TLS协议栈的使用进行安全性分析 -- the master secret retention problem

**分析对象** 

+ C语言核心：BoringSSL
+ Java代码层：Conscrypt and OkHttp

**分析步骤** 

+ 对于内存镜像进行黑盒安全性分析 --> 寻找可能被恢复的密钥
+ 对于BoringSSL、Conscrypt与OkHttp的实现代码进行深度分析 --> 定位Java代码层对于BoringSSL的*引用计数特性（reference counting feature）* 的错误使用 --> 阻止BoringSSL清理密钥

**结论** 

+ 问题长期被忽视，Android 4 - Android 8，存在漏洞
+ Android Chrome Application，特别有问题
<!--more-->
## 1. 介绍

**TLS**

+ PFS（Perfect Forward Secrecy）


+ Memory Zeroization
+ + 缓存公钥操作（computationally expensive）的结果 --> Session Resumption
  + 上层代码具有自己的密钥保留逻辑 --> 存在漏洞

**Android**

+ Memory Disclosure Attacks


+ Android应用程序具有复杂的生命周期 --> sleep & wake up --> 在进行残留密钥清理工作之前进入睡眠状态

## 2. 背景

**TLS**

![20180324133248](/images/2018-06-11/20180324133248.png)

**Session Resumption**

+ 重用之前TLS连接的master secret
+ session identifiers & session tickets

![20180324133303](/images/2018-06-11/20180324133303.png)

## 3. 黑盒安全性分析

![20180324143206](/images/2018-06-11/20180324143206.png)

**测试框架**

+ scriptable & automated

**测试**

![20180324145735](/images/2018-06-11/20180324145735.png)

**结论**

- NOT Effective --> 清除master secrets

![20180324152842](/images/2018-06-11/20180324152842.png)

**提出的问题**

+ master secrets的残留是否是程序性能与安全性权衡的结果？
+ 为什么改变并发连接的数量对残留master secrets的数量没有影响？
+ 为什么显式调用garbage collector与将应用程序切换到后台效果相同，均未能清除master secrets？
+ 为什么HTTPS API与底层的TLS API对于master secrets的清除行为不同？
+ 为什么使用默认的SSL Context比为每个连接创建新的SSL Context，master secrets在内存中残留的时间更久？

## 4. Android架构的深度分析

**分析对象**

+ BoringSSL (C), Conscrypt (Java), and OkHttp (Java)

**主要方法**

+ 标记代码库中的重要类、重要函数
+ 探究代码库之间的交互方式，以及代码库的内存管理实现

**概述**

![20180324192408](/images/2018-06-11/20180324192408.png)

![20180324185819](/images/2018-06-11/20180324185819.png)

**BoringSSL**

+ master-secrets存储于SSL_SESSION对象中，当没有其他对象引用该SSL_SESSION对象时，master-secrets被清除
+ C语言不提供对于SSL_SESSION对象的存活状态检测 --> manual reference counting
+ BoringSSL不支持client mode下的session resumption --> 由Conscrypt实现
+ 结论：BoringSSL对于*引用计数特性（reference counting feature）* 的使用没有错误

![20180325102954](/images/2018-06-11/20180325102954.png)

**Conscrypt**

+ 实现client mode下的session resumption --> maintain a session cache (Lazy Deletion)
+ 没有任何性能优势的问题代码 --> a bug, not a performance trade-off
  + 依赖GC（garbage collection）
    + the *finalize()* method
    + Memory leaks --> master keys survive after explicit calls to the garbage collection
  + session cache的LRU实现
    + default session timeout unnecessarily long
    + timeout not used to remove sessions from the session cache
  + SSLContext的单例模式
    + use a singleton pattern and manage globally
    + prevent objects from ever being garbage collected --> the default SSLContext never becomes garbage

**OkHttp**

+ ConnectionPool存储内部保存SSLSocket对象的Connection对象 (Eager Deletion)
+ deletion of connections using a Timer, 5-minutes timeout
+ 如果在Connection对象超时之前，应用程序被切换至后台，进入睡眠状态，GC（garbage collection）将不会被调用 --> background apps are never garbage collected

**问题总结**

+ 最大的问题：Conscrypt层与其他层的交互

+ 内存中残留的master secrets的数量是恒定的——session cache的大小 --> the connection pool’s size

  does not vary with the number of concurrent connections


## 5. 评估攻击的可行性

**威胁模型**

+ 前提条件
  + 攻击者能够被动捕获网络数据包
  + 攻击者掌握Android内存泄露漏洞

**提取master secret**

+ SSL_SESSION结构具有能够被轻易设别的signature pattern

![20180325102954](/images/2018-06-11/20180325102954.png)

**在实际使用中衡量master secrets的残留**

![20180325141555](/images/2018-06-11/20180325141555.png)

+ Chrome做出了一个性能和安全的权衡
+ YouTube的内存需求迫使Android为YouTube回收内存，清除过期页面
+ 攻击者拥有大量的时间来获取受害者手机内存中残留的master secrets

**解密TLS通信**

+ usernames, passwords, or HTTP session cookies

## 5. 讨论

**解决方案**

+ 解决Conscrypt与BoringSSL之间的对象管理冲突


+ Strawman solution：在session完成之后立即清理master sectets
+ Hooking Android's core framework：在应用程序生命周期变化时触发清理master sectet的例程
+ Concurrent eager deletion：使用另外一个线程及时清除session cache中过期的SSLSession对象

**观察和未来的工作**

+ Android在未来应该将Conscrypt与OkHttp两个代码库进行合并，实现更加积极的清理过程
+ 调整抽象的界限，将更多的工作上升至Conscrypt代码库中，或者将更多的工作下降至BoringSSL代码库中
+ 在Android Studio中添加额外的对于JSSE密码库的使用的检查
+ 安排一个以较低优先级运行的后台线程进行清理工作
+ 这种类别的攻击长期没有引起足够的重视，静态安全分析工具仅仅考虑只有一种编程语言的情况


