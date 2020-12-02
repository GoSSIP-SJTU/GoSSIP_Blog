---
layout: post
title: "PatternListener: Cracking Android Pattern Lock Using Acoustic Signals"
date: 2018-12-28 15:52:26 +0800
comments: true
categories: 
---

作者：ManZhou1,QianWang1,JingxiaoYang1,QiLi2,FengXiao1,ZhiboWang1,XiaofengChen3 

单位：School of Cyber Scienceand Engineering, Wuhan University, Graduate School at Shenzhen & Department of Computer Science,Tsinghua University, School of Cyber Engineering, Xidian University 

出处：CCS’18

论文：https://arxiv.org/pdf/1810.02242.pdf

<hr/>

#### ABSTRACT

手势密码的应用十分广泛，包括解锁手机或是应用内授权等。传统的攻击方式一般需要满足至少一条：

1. 物理上接近被攻击者
2. 控制被攻击者所在网络中的设备（例如，路由器）

本文提出了一种新的方法，称为PatternListener，能够利用受害者手机的扬声器和话筒完成攻击。当受害者输入手势密码时，PatternListener通过扬声器播放一种18kHz以上大部分人无法察觉的音频，并用话筒进行录音。通过分析输入手势密码时，手指动作对声波的干扰，对密码进行恢复。

<!--more-->

#### INTRODUCTION

调查显示，40%的受访者使用手势密码，其中33%使用手势密码进行应用内授权而非解锁手机。

但是，由于沙盒、TrustZone这样的安全机制，传统的攻击方式难以实施。因此，利用所有应用共享的硬件资源（传感器、摄像头、话筒等）的侧信道攻击开始受到关注。

之前的一些使用侧信道方式的攻击，多针对PIN码。这些攻击主要的问题在于，准确率低，易受到周围环境的影响。这些问题在针对手势密码的场景会更加棘手。

另外，由于一般的移动设备只有一对摄像头和话筒，已有的2D gesture tracking的方法不能被应用到PatternListener。

面对以上问题，PatternListener的主要贡献有：

1. 首先利用扬声器和话筒实施攻击，并提出了共享硬件资源可能存在的问题。
2. 利用不易察觉的音频实施攻击，健壮性更强，在实际攻击场景中所受到的限制更小。
3. 运用多种算法对反馈的样本进行分析，能够批量分析大量的样本。
4. 使用在售的手机进行测试，能够在五次尝试内破解90%（样本共130个）的手势密码。在分析过程中发现，更多的线条并不能增强安全性。另外PatternListener对于外界变化有很强的抗干扰能力。

#### CRAKING PATTERN LOCK

![](/images/2018-12-28/2.png)

PatternListener是一个运行在后台的恶意app，需要以下权限：扬声器、麦克风、动作传感器（包括加速度传感器、陀螺仪等）以及网络权限。

其中，只有麦克风权限是需要用户授权的，而很多应用都需要麦克风权限（Google Play上55%的社交app和52%的通信app都需要用户授权使用麦克风），因此PatternListener所需的权限都是易于获取的。

#### PATTERNLISTENER DESIGN

整个攻击分为四个部分：解锁检测、音频获取、预处理、手势密码恢复。

##### 解锁检测

一、screen unlock:

屏幕状态可以分为三种：non-interactive\pre-interactive\interactive

用户输入手势密码的阶段是pre-interactive完成之后，由于屏幕状态变化时，会产生一个广播，对于屏幕解锁的检测可以通过监听从non-interactive到pre-interactive的变化。

二、app unlock:

本文认为，用户在应用内输入手势密码之前有一定的行为特征。用户需要通过左滑或右滑找到某个应用，然后用户的动作会有一段时间的停顿，等待应用的加载。通过动作传感器，首先确定用户已经成功解锁手机，然后持续通过扬声器播放音频，检测是否出现左右滑动屏幕的操作。



##### 音频获取

一般手机的扬声器和麦克风的工作频率在50Hz~20kHz，而18kHz以上的高频声音大部分人听不见，即使一小部分能够察觉，只要使用更低的音量就不会被发现。因此攻击使用18~20kHz的音频。



##### 预处理

这一部分使用通信中常用的方法进行滤波，为了减少干扰使用了一些策略。

一、LEVD（Local Extreme Value Detection）算法:用于消除static component(环境中正常的噪声)。

取两个相邻的极大值和极小值的中间点，并用一个线性的算法来估计噪声的值。

极值点的选取应满足两个条件:

1. 新的极值点和上一个极值点之间的时间间隔大于门限值Ti(Ti等于两倍极值点时间间隔的均值)

2. 新的极值点和上一极值点振幅差值大于门限值Td(经验值)

二、TPI（Turning Point Identification）算法:用于将手势密码划分成笔画进行分析。

作者观察到，当出现图案绘制出现转折时，用户操作普遍有一段很短的停顿，这一部分的波形会变得平稳。因此，找到正确的turning point就能将得到的样本按照笔画分段。

分别对样本的正交部分和同相部分找到极值点（利用LEVD算法），然后对比两次得到的极值点，由于正交和同相的极值点应该交错出现，因此可以进一步过滤噪声（enviromental disturbance and hardware deficiency in C/O component）。

然后找出极值点时间间隔远大于平均极值点时间间隔的部分，即得到tuning point。

##### 手势密码恢复
![](/images/2018-12-28/3.png)

d1表示从L到M，移动中距离的变化量；d2表示从M到N的距离变化。用二元组（d1,d2）作为手指运动特征。

![](/images/2018-12-28/4.png)

![](/images/2018-12-28/5.png)

建立一个ground-truth data，根据上图中的公式，计算出样本值和集合中数据的相似度。

建立图形树如下图

![](/images/2018-12-28/6.png)

根节点的确定需要借助动作传感器来确认。

#### SYSTEM EVALUATION

收集了197个不同的手势密码，从中选出120种，并添加了调查显示用户最常用的10个手势密码，组成了测试集（共130个不同的手势密码）。从测试结果来看，该攻击对外界干扰的抵抗能力强，成功率高。对于较复杂的手势密码也有较高的成功率。

![](/images/2018-12-28/7.png)

![](/images/2018-12-28/8.png)


![](/images/2018-12-28/9.png)

(samsung C9正确率略高于HUAWEI P9是由于C9屏幕更大。)

#### COMMENT

提出了一种新颖的侧信道攻击方式，并能引发对共享硬件资源的保护和利用的思考。运用了许多不同的策略，成效显著。评估涉及全面。
