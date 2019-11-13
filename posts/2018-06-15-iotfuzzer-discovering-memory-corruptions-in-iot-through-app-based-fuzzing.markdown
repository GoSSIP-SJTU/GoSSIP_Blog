---
layout: post
title: "IOTFUZZER: Discovering Memory Corruptions in IoT Through App-based Fuzzing"
date: 2018-06-15 19:10:09 +0800
comments: true
categories: 
---


>论文链接：http://wp.internetsociety.org/ndss/wp-content/uploads/sites/25/2018/02/ndss2018_01A-1_Chen_paper.pdf

>slides链接：http://wp.internetsociety.org/ndss/wp-content/uploads/sites/25/2018/03/NDSS2018_01A-1_Chen_Slides.pdf

>作者：Jiongyi Chen , Wenrui Diao, Qingchuan Zhao , Chaoshun Zuo , Zhiqiang Lin , XiaoFeng Wang , Wing Cheong Lau , Menghan Sun , Ronghai Yang, and Kehuan Zhang 

>会议：NDSS 2018



## 一、概述

作者提出了一个新型自动黑盒Fuzz测试框架——`IOTFUZZER`，旨在监测物联网设备中的内存损坏漏洞

**主要优点**

- 无需获取IOT设备固件镜像
- 无需逆向分析
- 无需知道协议具体内容

**工作目的**

仅Fuzz测试，用于指导后续的安全分析，找出漏洞的根源。

**基于APP分析的Fuzz测试框架**

- 固件难以获取、解压
- 大多数IOT设备带有官方APP
- APP中包含丰富的命令（种子）消息，URL和加密/解密信息
- 轻量级，无需复杂的逆向和协议分析

<!--more-->

## 二、相关背景

![图1](/images/2018-06-15/01.png)

**IOT通信模型**

- 手机直接与IOT设备进行通信
- 使用供应商提供网络云服务进行通信

**工作中的挑战与解决方案**

- 不同设备可能使用不同的已知协议或未知协议，需要自动识别和Fuzz未知协议的协议字段
=> 在数据源处突变协议字段

- 处理加密的消息和提取密钥
=>运行时重用加密函数

- 未破解IOT设备，难以实时监控
=>心跳机制检测活跃度

## 三、框架实现

![enter image description here](/images/2018-06-15/05.png)

**1、UI分析**
目标是确定最终导致消息传递的UI元素。
执行静态分析以将不同活动中的UI元素与目标网络API相关联。
工具：**Monkeyrunner**


**2、数据分析**
识别协议字段并记录以协议字段为参数的函数。
动态污点跟踪硬编码字符串，用户输入或系统API。
工具：**TaintDroid**

**3、动态Fuzz**
hook目标函数，并传递突变参数。传递突变给多个挂钩函数，进行多次Fuzz它，并且可能会hook相同的函数以Fuzz多个协议字段。

![enter image description here](/images/2018-06-15/06.png)

随机选择一个字段的子集来进行变异，而不是突变所有字段（因为所有字段被突变的消息都可能被设备轻易拒绝）。

![enter image description here](/images/2018-06-15/07.png)

基于突变的Fuzz

 - 更改字符串的长度以实现栈溢出或堆溢出和越界。
 - 更改整数，双精度或浮点的值导致整数溢出和越界。
 - 更改类型或为未初始化的变量提供空值。

工具：**Xposed**

**4、实时监听**
>四种响应
> - 预期的响应
> - 意外的响应
> - 无响应
> - 断开
 
使用心跳机制，从IoT应用程序中提取心跳消息；在Fuzz测试过程中，每十次Fuzz测试插入一个心跳消息

## 四、评估与分析

**测试设备**
17台主流制造商提供的畅销产品， 所有这些设备都可以通过官方IoT应用程序通过本地Wi-Fi网络对设备操作。 通信协议和数据传输格式没有限制。

![enter image description here](/images/2018-06-15/08.png)

![enter image description here](/images/2018-06-15/09.png)

 **测试结果**
 通过使用`IOTFUZZER`自动化框架（每台设备运行24小时）对17台IoT设备进行Fuzz测试，在9台设备中发现15个严重漏洞（内存损坏）。包括：
 
- 5个基于堆栈的缓冲区溢出
- 2个基于堆的缓冲区溢出
- 4个空指针引用和4个崩溃

将在`IOTFUZZER`识别后进行一步检查。

 ![enter image description here](/images/2018-06-15/10.png)

**效率**
同`Sulley`和`BED`相对比，普遍具有较高的效率。

![enter image description here](/images/2018-06-15/11.png)

**UI性能分析**

应用相对简单，调用的事件和活动数量不多。

![enter image description here](/images/2018-06-15/12.png)

同时调用的函数不多，重要的函数被多次反复调用。

![enter image description here](/images/2018-06-15/13.png)

**Fuzz精度**

设备没有心跳响应或断开基于TCP的连接可能是由实验中的不可预知的通信错误引起的。因此精度普遍不高

![enter image description here](/images/2018-06-15/14.png)

## 五、测试案例

**基于WIFI的TP-LINK智能插头**

![enter image description here](/images/2018-06-15/15.png)

![enter image description here](/images/2018-06-15/16.png)

name字段本应是字符串，被Fuzz成整数。

当突变的消息传递到Wi-Fi智能插头时，设备将呈红色闪烁并拒绝任何有效的消息。

通过复杂的固件分析，这些漏洞是由使用未初始化的指针触发的。

![enter image description here](/images/2018-06-15/17.png)

**Belkin WeMo 开关**
![enter image description here](/images/2018-06-15/18.png)

如果没有提供命令消息“SetSmartDevInfo”的内容，则该消息会导致交换机崩溃并自动重启。

![enter image description here](/images/2018-06-15/20.png)

该消息触发`0x00000000`的无效读取访问。

![enter image description here](/images/2018-06-15/19.png)

如果没有，则数据结构中的指针未初始化，然后导致从`0x00000000`读取。



## 六、讨论与限制

**局限性**

 - 测试范围
    - 移动应用程序提供了便于设备管理的主要数据输入渠道，可以尝试使用其他数据通道（如传感器或调试端口）。
 - 连接模式
    - 目前仅通过Wi-Fi连接的设备，日后可扩展到其他通信模式，如蓝牙，Zigbee。
 - 结果判断
    - `IOTFUZZER`无法直接生成内存损坏类型和根本原因，在黑盒测试中，最终的漏洞确认通常需要进行一些手动操作。
 - 结果的准确性
    - 存在大量误报。
    - 在某些情况下，内存损坏不会导致崩溃。

