---
layout: post
title: "NativeX: Native Executioner Freezes Android"
date: 2020-11-17 15:40:32 +0800
comments: true
categories: 
---

> 作者：Qinsheng Hou ,Yao Cheng ,Lingyun Ying
> 
> 单位：QI-ANXIN Technology Research Institute，Huawei International
> 
> 会议：AsiaCCS 2020
> 
> 链接：[pdf](https://dl.acm.org/doi/abs/10.1145/3320269.3384713)

## 简介

Android的基本资源，包括系统资源和计算资源，都是framework层和native层的应用共享的。由于异步的工作机制，如果有一个进程独占了共享的资源，那么其他的进程就会因为等待资源被阻塞。为了避免这一问题，Android提供了一些资源管理机制，比如Application Not Responding (ANR)用于处理应用没有响应的情况，Low Memory Killer (LMK) 用于在内存少的时候杀死不重要的进程，watchdog用于重要系统服务没有响应的时候重启它。

但是，作者发现这些机制都不能很好的处理native进程对资源的独占情况。Android系统上运行的native进程包括两种：一是系统的Linux可执行文件，二是通过Java API `Runtime.getRuntime().exec()`和native API`fork()`和`system()`启动的。

作者设计了一个分析工具NativeX（Native eXecutioner）可以识别存在威胁的Android命令并自动生成PoC。在Android版本为4.2到9.0的设备上，作者发现这种漏洞存在于99.7%的超过20亿的IoT设备和手机。

<!-- more -->

## 攻击风险

1. 通过合法的Android命令（比如，`top` , `am`）占用重要系统资源导致系统冻结。
2. 通过产生大量native进程消耗计算资源。

## NativeX

![](/images/2020-11-17/uploads/upload_1e4ade77f1633e518611af0f1cb5fa80.png)

### 静态代码分析

输入是Android framework源码，输入是一系列文件系统资源。

1. 搜索所有重要的Android系统服务组成集合$\{S\}$。这些系统服务会在代码里注册Android watchdog，这样如果他们运行失败了会被watchdog重启。
2. 在系统服务里找到使用同步对象的method，它们会因为等待对象资源而被阻塞从而导致被watchdog重启。首先NativeX会通过找到`synchronized()`识别同步对象的集合$\{O\}$，然后在集合$\{S\}$搜索使用到这些对象的method，组成集合$\{M1_{direct}\}$，再提取这些method的caller组成集合$\{M1_{indirect}\}$，再加入他们的caller，最终生成一个集合$\{M1\}$。
3. 作者扫描了所有framework源码里使用对象$\{O\}$或者method$\{M1\}$的method，组成集合$\{M2_{tmp}\}$。在$\{M2_{tmp}\}$，作者有筛选了耗时多的method组成集合$\{M2\}$，包括两种method：
    - 循环里面有I/O操作
    - 使用到这10个method中任意一个：printCurrentState(), writeEvent(), dumpStackTraces(), printCurrentLoad(), addErrorToDropBox(), updateOomAdjLocked(), computeOomAdjLocked(), killProcessGroup(), isProcessAliveLocked(), updateLruProcessLocked()
4. 查找在集合$\{O,M1,M2\}$中使用到的文件系统资源$\{R\}$。

### 危险指令分析

Android指令的源码通常在/external/toybox/toys, /system/core/toolbox 和/frameworks/base/cmds下，作者提取源码，分析其中可以操作$\{R\}$里存储资源的命令。

### 自动化PoC生成器和验证器

生成器使用下面的模板生成PoC。

![](/images/2020-11-17/uploads/upload_0f7f32a426f42e8e4b6df67f386e1ff1.png)


NativeX 通过adb自动安装和启动PoC app，在10s后运行验证脚本（如listing2），如果设备在一段时间内（60s）没有反应，或者adb连接中断，就说明验证成功。
![](/images/2020-11-17/uploads/upload_9b3c20c302cf3b353cdb1ad65684eb8c.png)

## 评估

### NativeX评估

实验设备：19部手机，包括Google Nexus, Pixel和Samsung，华为, 涵盖4.2到9.0的6个主要的Android版本；2台电视盒子，系统版本为4.4和6.0。

作者在4.2到9.0Android系统上找到了表1里的被watchdog监视的系统服务。
![](/images/2020-11-17/uploads/upload_e7970d556a140e812b58afa95a8afd36.png)


表2是Android9.0上的系统服务以及NativeX找到的同步对象，直接调用/间接调用方法，进一步的找到了77个没有被监控的method集合$\{M2\}$，最终定位到了9个存储资源。/data, /dev, /mnt, /proc, /product, /storage, /sys, /system, /vendor。


![](/images/2020-11-17/uploads/upload_71ec2fdc68139e80bcf358ff9356ec70.png)
![](/images/2020-11-17/uploads/upload_ff9b419e133e464b499ebb05d42c1a5a.png)


作者通过Android watchdog的log信息对可以使用成功攻击的指令进行了分析。对于持续性指令（例如top）来说，导致系统重启的原因可能是1）通过互斥的操作占用了系统存储资源，导致重要的服务被阻塞。2）大量的线程耗尽了计算资源。而非持续性指令和不支持的指令，虽然不能独占系统资源，但是仍然可以通过这些指令fork大量的进程，在/proc存储中加入大量的目录和文件并消耗计算资源。

表4是使用这两种指令完成攻击需要的最少进程数。

![](/images/2020-11-17/uploads/upload_e7b7c5b39bd8e2b4af8de85dc2054091.png)


Android8.0和9.0会一直冻结到用户重启，之前的版本会自动重启。

### 攻击结果

图2是一直在计算AES加解密和进行攻击，充电状态下和未充电状态下的温度情况，测试设备是Google Pixel。

![](/images/2020-11-17/uploads/upload_e02971482f3845d22b4b61df32b3bacc.png)


图3评估这类攻击对电池的损耗。测试设备是Google Pixel 和Google Pixel 2.

![](/images/2020-11-17/uploads/upload_2acab98ece8a81996ad5175c3e80673f.png)


## 攻击影响及防御

### 实际攻击场景

1. 进行DoS攻击，作为勒索软件的一种实施方式。
2. 竞争对手的app运行时发起攻击，使用户卸载应用。
3. 使手机升温，物理伤害用户。

### 防御建议

1. 引入新的权限，管理执行文件的API `Runtime.getRuntime().exec()`，`system()`。
2. 限制手机上Native进程的数量。