---
layout: post
title: "The Android Platform Security Model"
date: 2019-07-25 18:53:47 +0800
comments: true
categories: 
---

作者：René Mayrhofer, Jeffrey Vander Stoep, Chad Brubaker, Nick Kralevich

单位：Google

出处：arXiv

原文：https://arxiv.org/pdf/1904.05572.pdf


<hr/>

#### 文章概述
安卓已经成为终端用户使用最多的系统之一，提供通信、娱乐、金融、健康等服务。其底层的安全模型需要为用户解决大量在复杂的使用场景中产生的安全威胁，同时取得安全、隐私与可用性之间的平衡。尽管安卓整体系统架构、访问控制机制等的底层设计原则都能在一些公开的资料中找到，但安卓的安全模型一直没有正式发表出来。所以在这篇文章中，作者对安卓安全模型进行抽象，并基于对威胁模型、安卓生态环境的定义来分析安卓的安全实现是如何缓解这些具体威胁的。
<!--more-->

#### 安卓背景

##### 1. 生态环境
安卓的部分设计选择必须考虑整个生态环境。如果安卓生态环境的主体(用户、应用开发人员、操作系统)对于某项操作无法达成一致，那么该项操作就不应该被允许(default-deny)。

+ 安卓是一个针对终端用户(end user focused)的操作系统。
    + 系统的可用性
    + Safe by default
    + 考虑非专业用户
+ 安卓生态系统体量很大。
    + 移动平台市场占有率(2018年：美国63%, 全球56%)
    + Hundreds of OEMs (Original Equipment Manufacturers)
+ 安卓App可以用任何语言编写。
    + Java APIs
    + 需要运行时保护

##### 2. 安卓安全原则
+ Actors control access to the data they create.
    + 文件系统
    + 内存
+ Consent is informed and meaningful.
+ Safe by design/default.
+ Defense in depth.


##### 3. 威胁模型
+ 攻击者可以物理接触安卓设备
    + T1: 攻击者完全控制关机设备
    + T2: 攻击者控制锁屏设备
    + T3: 合法的其他用户控制未锁屏设备
    + T4: 攻击者在设备附近(通过Wi-Fi, 蓝牙等)
+ 网络通信是不可信的
    + T5: 攻击者被动地监听流量
    + T6: 攻击者主动地篡改流量
+ 设备可能运行不可信的代码
    + T7: 滥用操作系统的API
    + T8: 利用操作系统的漏洞
    + T9: 滥用其他App提供的接口
    + T10: 不可信的来自Web的代码(JavaScript)
    + T11: 模仿系统或其他App的UI来欺骗用户
    + T12: 读取操作系统或其他App的UI内容
    + T13: 向操作系统或其他App的UI注入内容
+ 设备可能处理不可信的数据
    + T14: 处理不可信数据的代码被利用
    + T15: 滥用标识符进行定点攻击(如发送垃圾邮件等)

#### 安卓平台安全模型
1. 三方许可(用户、平台、开发者)
    任何行为都需要获取三方许可才能够执行，该规则覆盖主客体之间的操作。
2. 开放的生态环境
    生态系统不限于某一单独的应用市场，App的开发者可以自己定义App之间交互的API。
3. 安全是一个兼容性(compatibility)的需求
    安卓设备的规范(Specification)定义在CDD (Compatibility Definition Document)中，其中也定义了一些安全要求。设备制造商必须遵守CDD中的规范，只有满足CDD条件，同时能够通过CTS (Compatibility Test Suites)的设备才能够成为安卓设备。例如，Root设备在理论上是不属于安卓设备的，因为它违反了CDD中对于沙盒、隔离机制的要求。
4. 恢复出厂设置能够将设备恢复为安全的状态
5. 应用程序是安全主体(principals)


#### 实现
##### 1. 许可(Consent)
+ 开发者: 对App或App数据的操作需要得到开发者的许可，从而防止被设备上其他App注入代码、窃取数据。
    + 反调试
    + App签名
+ 平台: 决定什么操作是允许的
    + Verified Boot (code signing)
+ 用户: 用户需要明白一些行为的意义，并为其提供许可，但这往往是整个系统中最困难的部分，需要操作系统为用户提供提示。
    提示原则：
    + 避免过多的提示
    + 用户需要理解提示
    + 细粒度许可
    + 操作系统不应当将困难的问题交给用户
    + 为用户提供撤销决定的方式

##### 2. 认证
安卓平台用户认证的主要方式是通过锁屏，而锁屏的设计需要在安全和可用性之间做权衡。一方面，用户平均一天约50次解锁手机进行短时间操作(10~250秒)，因而锁屏妨碍用户与设备的无缝交互。但另一方面，从安全角度来说，锁屏是不可或缺的。
安卓9.0使用分层认证模型:
+ 主要认证模式: PIN，密码，图案
+ 次要认证模式: "强"生物信息
+ 第三认证模式: "弱"生物信息和其他模式(如蓝牙等)

##### 3. 隔离与封锁
+ 方式:
    + 访问控制 (e.g. default deny)
    + 减小攻击面 (i.e. principle of least privilege)
    + Containment
    + Architectural decomposition
    + Separation of concerns

+ 权限
    + Discretionary Access Control (DAC)
    + Mandatory Access Control (MAC)
    + 安卓权限
+ 应用沙盒
    安卓系统为每一个App分配不同的UNIX user ID (UID)以及一个私有目录。在此基础上安卓增加了很多沙盒的限制，包括MAC策略、运行时权限等。
![](/images/2019-07-25/1.png)
+ 系统进程沙盒
    安卓系统也为一部分系统进程分配UID。
![](/images/2019-07-25/2.png)
+ 内核及其之下的沙盒
    + Keymaster
    + Strongbox
![](/images/2019-07-25/3.png)

##### 4. 本地数据加密
安卓5.0以后引入了全盘数据加密功能(Full Disk Encryption, FDE)，对整个用户数据进行加密。紧急通话、闹钟等功能也需要用户输入密码，否则无法使用。
安卓7.0以后引入了文件加密(File Based Encryption, FBE)，密钥保存在TEE中，支持文件粒度的加密。同时提供Direct Boot功能，保证紧急通话、闹钟等功能即使在用户未输入密码的情况下也能正常使用。


##### 5. 传输数据加密
安卓假定所有网络都是不可信的，因而所有流量都需要进行端到端加密(TLS)。
![](/images/2019-07-25/4.png)
##### 6. 漏洞缓解
大约85%的安卓安全漏洞都与不安全的内存访问相关，最有效的解决方案是使用Java等内存安全语言。同时，Android中也增加了ASLR，RWX限制等内存保护措施。


##### 7. 系统完整性
从安卓4.4开始引入Verified Boot (Linux dm-verity)，并在安卓7.0之后强制实施。

##### 8. 补丁
因为安卓碎片化情况的存在，安全漏洞很难在短时间被完全修复。从2015年8月起，安卓开始每月发布漏洞公告和补丁，并在安卓8.0后引入Project Treble，试图改善安卓碎片化的情况。

#### 特殊情况

+ 获取安装的所有App包名
+ VPN应用可以监视/阻塞其他App的流量
+ 备份
+ 企业版本
+ 恢复出厂设置


