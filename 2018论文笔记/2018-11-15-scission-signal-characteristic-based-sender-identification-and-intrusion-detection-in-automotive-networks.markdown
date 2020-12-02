---
layout: post
title: "Scission: Signal Characteristic-Based Sender Identification and Intrusion Detection in Automotive Networks"
date: 2018-11-15 15:24:48 +0800
comments: true
categories:
---
作者: Marcel Kneib、Christopher Huth

单位: Bosch Engineering GmbH

出处: CCS'18 

原文: https://dl.acm.org/citation.cfm?id=3243751

随着汽车可连接性的增强，攻击面也越来越多。这种攻击不仅对汽车本身造成破坏，也可能进一步危害人身安全。CAN协议由于设计之初并没有考虑安全性，比如对发送者的身份进行认证，验证消息的完整性等等，这就造成了一旦连接在CAN总线上的ECU被攻击者恶意控制，攻击者就能任意伪造消息。本文，来自博世的工程师设计了Scission系统，用CAN报文的物理特性来判断消息的合法性。
<!--more-->

## 介绍
由于汽车的开放性和高连接性，汽车安全已成为一个主要问题。假如攻击者可以物理访问ECU，就能控制很多安全敏感的功能，并且无视驾驶员的控制，比如停止发动机或停止制动。由于要求进行物理访问是不现实的，有研究者对汽车进行了系统的分析并利用其他攻击面，包括远程连接，例如蓝牙和蜂窝网络。使用这些远程攻击，使得对ECU的攻击变得更加现实。Miller和Valasek展示的在Jeep Cherokee上的攻击，导致140万辆汽车被召回。他们通过蜂窝网络访问车内的网络，进而控制引擎，以及刹车和转向等。研究表明，CAN仍然是当下使用最广泛的用于车载系统的总线协议，可以在今天的每辆车中找到。但是大多数车载系统中没有实现足够的安全措施，攻击者可以发送伪造的CAN消息到其他设备，CAN的接收者无法识别发送者，从而无法验证消息的真实性。

## 背景
### CAN
CAN是一种车用总线标准，于1983年在Robert GmbH设计。 它使用两根绞合线（高，低）把多个ECU连接起来，两端用120Ω电阻端连接。低速CAN信号在传输显性信号（0）时，驱动CANH端抬向5V，将CANL端降向0V。在传输隐性信号（1）时驱动CANH和CANL为2.5V。CAN帧的信号如图所示，其中高电平为蓝色，低电平为红色。

![](/images/2018-11-15/media/15414929550193/15414945633417.jpg)

数据通过CAN帧传输，每个有效载荷能够承载8个字节。数据帧用唯一标识符（一般是11bit）来表示数据的优先级和含义。注意没有节点地址存在。由于CAN是广播总线，因此多个总线参与者可以同时访问总线。显性信号屏蔽隐性信号，也就是说，标识符越小，优先级越高。

一些研究者发现，CAN报文的物理特性可作为识别发送者身份的依据。比如说电阻的阻值，一般来说有5%的容差，那么这个容差的大小就可以作为相应的ECU的一个特征被识别出来。并且这种物理特性可以保持一定的时间不变，同时攻击者如果想影响该属性的话，必须通过物理接触的手段。

作者选取物理特性作为指纹：
* 电源电压
* grounding
* 电阻，导线
* 总线拓扑的瑕疵

## 相关工作
* CAN+ protocol，即通过增加MAC，这需要软硬件兼容这种新协议，作者首先否定了这种方案
* additional messages，也是通过增加MAC，只需更改协议，但对总线负载提出了更高的要求
* truncated MACs，是一种折中的方案，即取MAC的一部分，为了避免冲突至少使用64bit ，并且密钥管理和维护计数器都需要额外的算力

作者同时指出，通过增加MAC的方式只能防止消息被篡改，但伪造的消息依然可以发送。

* Clock-based Intrusion Detection System，只能识别周期性的CAN报文，已被研究者证明不安全
* CAN信号的特征可以识别origin，并且这个特征很长一段时间不变
* 使用extended identifier field的18bit 来识别消息的origin，但需保证这extended不被用作别的用途，同时也许重新设计ECU使得其支持扩展的identifier field
* Voltage-based attacker identification，只使用了数个CAN报文的电压平均值作为特征，不够稳定

![](/images/2018-11-15/media/15414929550193/15415959578765.jpg)

## SCISSION
###  Security and Threat Model
不同的车上的ECU型号不同，所构成的网络也各不相同。根据ECU的功能可以分为powertrain, comfort 和 body。如图，展示了两种示例架构。Scission使用一个附加的ECU接到总线上，来监控总线。为了防止Scission被bypass，假设该ECU是被安全的实现的（即可信的）。对于整合进入系统的ECU，网关可以确定收到的消息是否从合法的ECU发送。

![](/images/2018-11-15/media/15414929550193/15417459559808.jpg)

CAN没有提供验证机制，因此每个总线参与者都能够使用所有可用的标识符并且接收方无法验证消息是否来自合法的发送者。 为了弥补这个缺点，Scission根据CAN数据包的物理特性来确定收到的数据包的身份。所以，只有当被控制的ECU或者额外的的ECU发送伪造的信号时，Scission才能检测到攻击者。

* Compromised ECU，作者认为最容易被攻击的ECU是那些带有连接能力的，比如 cellular, WiFi 和 Bluetooth。因为这样，攻击者就能远程发送CAN数据包。
* Unmonitored ECU，针对passive 或者 unmonitored device，比如利用更新机制插入恶意代码
* Additional ECU，攻击者加一块ECU，当然这需要物理接触汽车，典型的比如通过OBD-II获取车的状态信息
* Scission-aware Attacker，攻击者试图误导IDS，比如通过消耗电池，或者使ECU所处的环境的温度发生变化。

### Fingerprinting ECUs
如图4所示，首先对信号进行采样，然后进行预处理，接收的数据被分成单独的比特并根据特征排序。随后，提取时域和频域的不同特征。这些特征，就是了实际的指纹。 系统启动的时候，训练这些指纹的模型，然后将其用于分类计算概率。在之后，系统监控发送的数据，当检测到攻击时触发警报。

![](/images/2018-11-15/media/15414929550193/15417501484059.jpg)

识别指纹的步骤可以分为如下几步：
* Sampling
  采样率20M/s，取高低电压的差值作为原始数据
* Preprocessing
  对原始数据分组，根据波形分成显性上升沿，显性非上升沿和隐形下降沿
  ![](/images/2018-11-15/media/15414929550193/15421634482828.jpg)

* Feature Extraction
  对每组数据**分别**计算最大值，平均值，方差等等。。
  Relief-F，特征选择算法
  ![](/images/2018-11-15/media/15414929550193/15421634644127.jpg)

* Model Generation and Classification
  Logistic Regression，分类算法，有监督的学习。
  那么为什么是有监督的学习？下面给出了解释。

* Deployment and Lifecycle
  系统必须要保证学习的阶段不被干扰，所以这一阶段最好是放在整车厂进行，那如何保证之后学习的参数不被因为时间的变化准确度下降？作者提出需定期对车进行检修，或者使用在线的机器学习算法，（那么在线的话学习的数据会不会被污染？这个时候就要用到加密）

### Intrusion Detection using Fingerprints
*  Detecting Compromised ECUs
  根据训练的分类模型，每当接受到一个新的数据包就预测其合法的概率。当数据被标记为未被识别ECU时，就有很大的概率产生误报。
  如何防止误报？tmax记录每个ECU允许发送的异常数据包的概率的阈值
  那么会不会导致漏报？tmin记录每个ECU应该发送的正常的数据包的概率的最低阈值

*  Detecting Unmonitored ECUs
*  Detecting Additional ECUs
  对于Unmonitored和Additional的检测，系统会为每个ECU记录一个全局的counter，当检测到可疑的数据包counter就会+1，正常的数据包会-1。对于Unmonitored，检测出ECU不应该发出不属于其的报文很容易。对于Additional ECUs，因为加了一块ECU，会导致bus的拓扑改变，因此每个ECU都会被影响，counter会迅速增加。

*  Detecting Scission-aware Attacker
  在学习阶段，Scission是没有作用的，因此必须严格保证学习阶段不被干扰。

## 评估
作者最终实现了很高概率的识别接收到的CAN数据包的发送者，并且根据这些特征识别compromised, unmonitored 和 additional ECUs。作者分别在一个原型系统，Fiat 500， Porsche Panamera S E-Hybrid上做了测试。

### Fingerprinting ECUs
每个ECU的前200个frame作为训练集。在这部分，作者使用了5个Arduino Uno作为CAN的原型，每个Arduino驱动两块CAN shield。每条总线的两端连上电阻，双绞线。连接同一电源。
![](/images/2018-11-15/media/15414929550193/15417529252884.jpg)

Prototype，混淆矩阵，每一列是收到的数据能够用指纹正确区分的概率。
![](/images/2018-11-15/media/15414929550193/15417532051183.jpg)

Fiat 500，真实的测试，有6个ECU，为了能发测试数据，接了两块树莓派作为ECU
![](/images/2018-11-15/media/15414929550193/15417533791713.jpg)

Porsche Panamera S E-Hybrid，真实的测试，同样又接了两块树莓派
![](/images/2018-11-15/media/15414929550193/15417535110653.jpg)


### Detecting Compromised ECUs
![](/images/2018-11-15/media/15414929550193/15417535803025.jpg)

Prototype，用两块Arduino发送伪造的数据包，98%的攻击识别率
Fiat 500，用两块Raspberry Pis，100%的攻击识别率
Porsche Panamera S E-Hybrid，96.82的攻击识别率

### Detecting Unmonitored ECUs
只在Fiat 500上做了实验，在不识别ECU7的前提下，在200条可信的数据包里插入从ECU1发出的伪造的数据包，异常包加4，正常包减1，阈值200
![](/images/2018-11-15/media/15414929550193/15417537977590.jpg)


### Detecting Additional ECUs
Prototype，先记录没有ECU9的数据包，并用这些数据训练，在420个正常的数据包后，发送恶意的数据包，在第53个恶意数据包发出后，ids检测出异常
Fiat 500，在发送390个正常的数据包后，发送恶意的数据包，在第77个恶意数据包发出后，ids检测出异常
Porsche Panamera S E-Hybrid，在发送370个正常的数据包后，发送恶意的数据包，在第79个恶意数据包发出后，ids检测出异常
![](/images/2018-11-15/media/15414929550193/15421643400462.jpg)


## 总结
作者认为在车载网络中使用IDS是一种很有前途的技术。作者提出了一种从CAN信号中提取指纹的方法识别收到消息的发送者的方法。评估显示，Scission能够正确识别的概率为99.85％。他们所提出的IDS也能够检测到来自未受监控和其他设备的攻击。
