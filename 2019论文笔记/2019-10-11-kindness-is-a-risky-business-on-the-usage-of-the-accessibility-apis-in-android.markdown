---
layout: post
title: "Kindness is a Risky Business: On the Usage of the Accessibility APIs in Android"
date: 2019-10-11 01:53:18 -0400
comments: true
categories: 
---
作者：Wenrui Diao  , Yue Zhang  , Li Zhang  , Zhou Li  , Fenghao Xu  , Xiaorui Pan k , Xiangyu Liu , Jian Weng  , Kehuan Zhang  , XiaoFeng Wang 

单位：Shandong University，Jinan University, University of California, The Chinese University of Hong Kong, Indiana University Bloomington, Alibaba

会议：RAID2019

链接：[pdf](www.usenix.org/system/files/raid2019-diao.pdf)

<hr/>

## Introduction

Accessibility API指的是帮助有困难的用户使用设备的API。现在的主流操作系统都提供了accessibility service，使用这个服务必须要申请BIND_ACCESSIBILITY_SERVICE权限，确保只有只有系统能绑定这个服务。

之前的研究局限于有BIND_ACCESSIBILITY_SEVICE权限的恶意应用可以进行的攻击，但是他们没有对Android accessibility framework的底层实现进行系统的研究。

这篇文章里作者会对Accessibility API的使用及其支持的architecture进行系统的评估。通过代码评估和大量的app的扫描，作者发现了大量的API被误用的情况。作者总结了Android accessibility API框架设计上的缺点。针对accessibility的设计作者提出了两种攻击方式：installation hijacking和notification phishing。作者也提出了相应的缓解方法。
<!--more-->

## Background

assistive app：提供accessibility服务的app，实现时继承类AccessibilityService。

accessibility服务框架有三个部分：topmost app，system-level服务AccessibilityManagerService，有定制的accessibility服务的assistive app。

简单的accessibility服务调用流程：

![img](/images/2019-10-11/fig2.png)

1. 创建和发送事件。由view生成AccessibilityEvent，属性包含EventType , SourceNode , ClassName , and PackageName 等用于描述当前view的状态。通过binder IAccessibilityManager发送给相关各方。

2. 分发事件。经过事件类型等确认后，中心化的Manager AccessibilityManagerService通过binder IAccessibilityServiceClient分发给所有绑定的服务。

3. 接收和绑定事件。通过回调函数onAccessibilityEvent，assistive app进行事件处理。如果需要注入action，会回溯寻找包含事件的源直到定位到topmost app。

   

## Accessibility API 使用

作者从Google Play上获取了91,605个APK，囊括了除了游戏分区之外的所有类别最受欢迎的App。

![img](/images/2019-10-11/fig3.png)

**Q1 Accessibility服务的使用**

在manifest文件中搜索BIND_ACCESSIBILITY_SERVICE。共有337个assistive App（0.37%）提供了342个accessibility服务，半数以上的app（56.7%）有百万以上的安装。

**Q2：Accessibility Capabilities的使用**

在configuration文件搜索键值对。如Listing1中的[ android:canRetrieveWindowContent="true" ]。

![img](/images/2019-10-11/listing1.png)

Q1中发现的342个服务中有8个没有从configuration中获取到，可能是1）虽然声明了但是没有实现；2）因为加了一些保护是的分析失败。在图4结果中，C0是default capability，有128个服务只使用了这个。

![img](/images/2019-10-11/fig4.png)

**Q3：使用的目的**

作者发现Google Play上合法的App都会提供为什么要使用accessibility服务的描述。

![img](/images/2019-10-11/listing2.png)

1.爬取服务描述。从res/values/strings.xml之类的路径提取，如果不是英文就用Google Translate API转换。

2.Part-of-Speech Tagging。使用spaCy进行标记，提前收集所有的动词。

3.语义关系提取。从语义关系树中提取出[obj+action]关系，这一步不会加入否定句里的动作，比如（never） collect private information"。

4.匹配和分类。由于机器学习在短句中的分类效果并不好，这里分类的方法是作者设定的分类规则。比如kill进程的关系是：

[v kill n app ; v stop n app ; v block n app ; v kill n applic ;v stop n applic ; v block n applic ; n batteri ; n cach ; n acceler ; n power ]

作者会在进行分类的时候对规则进行调整，分不了就加入新的规则，分错了就调整规则。

![img](/images/2019-10-11/fig5.png)

结果中除了8个没有找到的service之外，13个没有提供描述，58个App是uncategorized指描述中没有有用信息。

![img](/images/2019-10-11/fig6.png)

作者发现只有11个app在描述中提到了是为了帮助残疾人。因为accessibility API设计的目的是为不方便的用户提供服务，所以除了这种目的的使用其他都属于误用。

![img](/images/2019-10-11/tab1.png)

Kill background process。权限KILL_BACKGROUND_PROCESSES只能kill进程但是不能防止他再启动，而FORCE_STOP_PACKAGES不能用于第三方APP。

Detect foreground apps。由于这个信息比较敏感，所以Google将权限GET_TASKS替换为一个系统权限REAL_GET_TASKS，使得第三方不能使用。

## 缺陷

1.对accessibility API的使用目的没有做限制。所有有相应权限的App都能调用。

2.对accessibility事件处理的完整性没有强保证。accessibility的框架是由事件驱动的，运行的逻辑完全依赖于接收到的事件。

3.对生成的accessibility event属性没有限制。没有任何权限的app也能生成事件。原本事件只应该由最上层的view发送，但是没有做强制性限制。

![img](/images/2019-10-11/listing3.png)

## Attack

### installation hijack

第三方的应用市场原本是不能进行静默安装的，必须由用户点击确认进行安装。它们为了方便用户，会使用accessibility API自动点击INSTALL按键。应用市场使用Intent加载下载的APK，Android系统会调用PackageInstaller，应用市场通过AccessibilityEvents检测UI，当显示安装确认页面时，应用市场调用accessibility服务点击INSTALL按键。

![img](/images/2019-10-11/fig7.png)

对于将下载的应用存储在外部存储空间的第三方应用市场，恶意应用只需要申请READ_EXTERNAL_STORAGE权限就可以进行劫持。检测到APK的下载后，使用一个相同的Intent去加载恶意的APK，系统相应一个installer，应用市场会检测到这个installer和界面的变化后会发起accessibility服务进行安装。

![img](/images/2019-10-11/tab2.png)

### Notification Phishing

![img](/images/2019-10-11/fig8.png)

部分用户觉得原始的status bar不好用，会下载第三方的status bar，例如Super Status Bar。在Android4.3之前，第三方的app不能直接监听notification，必须借用accessibility服务的功能。恶意的应用就可以发送虚假的事件给accessibility服务，事件的属性中包括notification和它指向的activity。
