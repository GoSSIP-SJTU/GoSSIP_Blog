---
layout: post
title: "Shattered Chain of Trust: Understanding Security Risks in Cross-Cloud IoT Access Delegation"
date: 2020-09-27 14:23:19 +0800
comments: true
categories: 
---

> 作者： Bin Yuan, Yan Jia, Luyi Xing, Dongfang Zhao, XiaoFeng Wang, Deqing Zou, Hai Jin, Yuqing Zhang 
>
> 单位：Huazhong Univ. of Sci. & Tech. , Indiana University Bloomington, National Engineering Research Center for Big Data Technology and System, Cluster and Grid Computing Lab, Services Computing Technology and System Lab, Shenzhen Huazhong University of Science and Technology Research Institute, Xidian University, National Computer Network Intrusion Protection Center, University of Chinese Academy of Sciences
>
> 会议：Usenix Security 2020
>
> 链接：  https://www.usenix.org/system/files/sec20-yuan.pdf 

一般情况下，IoT设备在供应商指定的云平台下进行管理，设备和用户也可以通过云来进行连接交互。

目前，一些主流的IoT云平台还支持跨供应商的设备访问管理， 例如Philips和SmartThings云支持委派设备访问另一个云（Google Home），这样用户就可以通过Google Home来管理来自不同供应商的多种设备。 

作者发现在缺少标准化的IoT委派协议的情况下，IoT云委派机制存在普遍的安全缺陷。本文系统化的分析了IoT访问授权委派机制的安全性，开发了一个半自动化的验证工具VerioT，结果表明几乎所有的委派机制都存在安全问题，可能会导致设备的未授权访问和设备冒充；为此作者实施了端到端PoC攻击来对这些问题进行确认，并进行了相应的评估。

<!-- more -->

### 1 Cross-cloud IoT Access Delegation

#### 1.1 Cloud-based IoT access

用户直接通过云访问设备，主要流程：

（1）设备用户将设备注册到云平台（device vendor cloud）上

（2）当用户访问设备时，云端对用户进行认证，然后将其指令发送给指定设备

#### 1.2 Cross-cloud access delegation

随着设备和供应商的数量急剧增加，为了便于管理，跨云IoT委派机制被提出并实施，来为不同的供应商的多种设备提供一个同一的管理接口，如图1所示。

例如， Google Home想要访问SmartThings云上的一个智能门锁，需要先从SmartThings Cloud上获得一个访问token（例如OAuth），SmartThings需要将访问设备的权限通过OAuth或其他方法分配给Google。 

- 用户登录到Google Home控制台（例如手机应用）
- 输入她的SmartThings账户凭据
- 如果通过认证，SmartThings会生成一个access token并将其传递给Google Home

![](/images/2020-09-27/fig1.PNG)

**Delegation chain**

IoT委派机制的主要流程如下：

1. 设备注册之后，设备供应商云平台（device vendor cloud）具有对设备的访问权限
2. 供应商将访问权限分配给delegatee cloud，实现不同云平台对用户设备的控制
3. delegatee cloud可以进一步将访问权限分配给其他云，平台上被授权的用户也可以进一步进行权限分配

**Delegation mechanisms**

1. OAuth机制
2. 自定义授权机制

#### 1.3 Security Requirements

- 安全且一致的委派机制：IoT委派机制会涉及到不同的厂商，需要满足委派链上各个参与方的安全需求。
- 不可绕过：委派策略应当要阻止所有未授权访问。
- 具有传递性：如果某个delegatee参与者规定了一项策略，那么委派链上所有位于其下游的参与者都应执行该策略。

#### 1.4 Threat Model

两类角色：1）管理员administrator：设备拥有者；2）委派用户delegatee user：权限可能会被吊销或到期。

- 两类角色均能够将设备的访问权限进一步分配给其他用户（delegatee user）

- 本文假定管理员和IoT云平台是诚实的，而委派用户可能是恶意的
- 恶意用户能够获取凭证和其他有用信息，例如发出请求、从日志文档中提取信息、捕获流量等
- 恶意用户无法窃听其他各方之间的通信

### 2 Security of Cross-Cloud IoT Delegation

 作者分析了10个主流的IoT云的跨云访问授权的安全性，总结了五种缺陷，并将其分为两大类。

#### 2.1 Inadequate Cross-Cloud Coordination

 **Flaw 1: Device ID disclosure **

场景：SmartThings想要将设备委派到Google Home上，那么它需要提供OAuth token和设备信息给Google，这些信息会传递给用户来使其控制设备。

缺陷：

- SmartThings使用设备ID作为触发-响应管理的认证token，设备ID是唯一固定的
- Google将设备ID公开给具有临时访问权限的用户，存在安全隐患

一个攻击场景：在Google Home上，如果 Airbnb的管理者将设备的权限赋给一位游客，那么SmartThings的设备ID就会永久暴露。即使之后游客的权限被Google Home撤销，该游客仍然知道设备ID，可能会进行设备事件伪造等攻击。 

![](/images/2020-09-27/fig2.PNG)

**Flaw 2: Leaking secret of delegatee cloud**

场景：在SmartThings上，delegator需要上传一个SmartApp的软件模块到SmartThings平台上，来帮助执行委派协议，管理设备的访问权限。例如，IFTTT云会通过分享一个秘密的URL来实现对设备访问权限的委派，当SmartThings上报一个事件时，会触发IFTTT云上的一个小程序，通过预先指定的规则来控制IFTTT云上的设备，如图3所示。

![](/images/2020-09-27/fig3.PNG)

缺陷：

-  通过IFTTT SmartApp提供的API，SmartThings用户可以获取秘密URL。，该URL是固定的。

一个攻击场景： 在SmartThings上，一旦Airbnb的管理者将设备的权限赋给一位游客，那么IFTTT的秘密URL就会永久暴露，这个游客就可以在之后直接与IFTTT设备进行通信。 

#### 2.2 Inadequate Policy Enforcement 

**Flaw 3: Exposing hidden devices in the delegator cloud**

场景：LIFX是一个IoT设备供应商，如果委派SmartThings来管理设备，则SmartThings需要运行LIFX SmartApp。

![](/images/2020-09-27/fig4.PNG)

在SmartThings上，可以将用户能够访问的设备定义为一个组，称作location，其中也包含与设备关联的SmartApp，location是SmartThings设备委派的最小单元；如果管理员想要授权某个location的设备，他需要将该location的控制权赋予delegatee user。

 LIFX SmartApp可以授权用户仅能访问设备的子集。

缺陷：

- LIFX SmartApp在SmartThings云上没有得到正确的保护，SmartThings上的授权用户可以从要给location的私有存储中读取信息。如图4所示。

**Flaw 4: OAuth pitfall**

场景：Tuya云采用标准的OAuth协议来委派Google Home对设备进行管理控制：Google Home上的用户输入Tuya凭据，如果检测通过，则Tuya会将其OAuth token转发给Google Home。

缺陷：

- Tuya云使用的OAuth方案不满足IoT委派机制的可传递性要求。

  Tuya云分发的设备访问OAuth token不代表用户，而代表Google

一个攻击场景： 如果用户在Tuya云上的访问权限被撤销，他仍可以使用其OAuth token通过Google Home来访问设备，如图5所示。

![](/images/2020-09-27/fig5.PNG) 

**Flaw 5: Abusing cross-cloud delegation API**

场景：

如图6所示，Philips Hue允许被授权用户通过手机应用访问Philips Hue网桥：

- 首先按下设备上按钮，开启绑定过程
- Philips应用通过本地网络从设备自动获取一个秘密token，称作whitelistID
- 用户登陆其Philips应用来从Philips云端获取OAuth token

有了这两个token，用户就可以通过Philips云来访问Hue网桥。云平台检查OAuth token，然后转发这些命令到设备，由设备检查whitelistID。如果撤销用户的访问权限，管理员只需要在云控制台上删除用户的whitelistID，即删掉设备上的whitelistID，这样用户的命令会被设备拒绝。 

Philips云使用一个API接口将设备访问权限授予另一个IoT云，用户在委托云中输入其Philips凭据，然后调用该API，这会返回OAuth token以及由设备生成的新的whitelistID。这样，委托云就可以向Philips Hue云发出命令来操控设备。 

缺陷：

- 在撤销权限时，管理员会删除whitelstID，但委派用户的帐户仍保留在由Philips云维护的设备访问列表中。

一个攻击场景：在权限被撤销后，用户可以重新调用API接口，获得到新的whitelistID和OAuth token，仍然可以访问 Philips Hue网桥。

### 3 System Modeling and Formal Verification 

#### 3.1 Overview 

作者设计并实现了一个半自动化工具——VerioT，来检查真实世界中IoT云委派机制的缺陷。

![](/images/2020-09-27/fig7.PNG)

**主要架构**

VerioT包括3个核心组件：模型生成器model generator，模型检查器model checker和反例分析器counterexample analyzer，如图7所示。 

- 由于不同的云通常支持不同的委派操作，很难构建出可以描述任何形势的模型。因此，作者针对可能发生委派操作的每组特定的真实世界云平台进行建模，并将这些云称为委派设置dele-setting。

- 模型生成器为每个dele-setting生成模型。以配置文件作为输入，包括dele-setting中的参与者（ delegator and delegatee clouds, user, device ）和参与者所支持的委派操作。
- 模型生成器生成的模型会通过模型检查器来进行检测，验证预定义的安全属性。 
- 模型检查器报告一个反例，这代表可以找到一种状态，该状态具有跨系统角色的访问路径，从而使未经授权的用户可以访问设备。

#### 3.2 Results

作者使用VerioT评估了10个主流的IoT跨云访问授权的安全性，发现了6类新的授权缺陷，影响了全部的10个云平台。作者手工确认了所有缺陷，并且使用真实的设备实现了五类缺陷的端到端PoC攻击。

**Dataset**

5种设备供应商云：Philips Hue，August，LIFX，Mi-Home，iHome

5种委派云：Google Home，Samsung SmartThings，IFTTT，Amazon Alexa，Wink

**Measurement**

（1）Prevalence of Vulnerable Delegation

测试中的所有云都存在不安全的委派机制。

（2）Scope of Impact

- IoT clouds affected by Flaw 1

  如上述所示，Google Home公开了其委派云（即SmartThings）的设备ID。未经授权的委派用户可以利用获得的设备ID来模拟设备事件。

  作者手动检查了其他云，发现其中三个将设备ID用作秘密令牌：SmartThings，TP-Link Kasa和elinkSmart，会导致Flaw 1，甚至可以通过触发规则控制其他设备。

- IoT clouds affected by Flaw 2

  所有授权访问IFTTT的供应商都可能受到影响，作者检查了34个设备供应商会受到影响。

- IoT clouds affected by Flaw 3

  SmartThings云泄漏了其他delegator云存储在其SmartApp中的凭据。因此，任何在其SmartApp中存储敏感信息/令牌的云都会受到Flaw 3的影响。作者检查了127个授权访问SmartThings的设备供应商的开源应用，发现18个SmartApp存储了敏感信息。

- IoT clouds affected by Flaw 4

  如前所述，Tuya云的IoT委派机制存在安全风险。作者发现这会影响了至少58个IoT设备供应商，这是由于Tuya云不仅自己制造设备，还向其他没有云的设备制造商提供云服务。

- Conﬂicting security policies across clouds

  为了提供跨云委派服务，主流的云平台都要求提供用户设备信息，包括设备ID，名称，型号，版本和类型。但是它们都没有描述其安全性规定：如何处理数据，是否安全敏感等。这表明目前IoT云之间还缺乏统一的安全管理。

### 4 LIMITATION

- 配置文件的生成复杂，需要耗费较长时间，且如果没有相关标准且明确的IoT云访问授权协议，则无法了解该信息。 
- 不能覆盖所有的IoT云委派机制，仅针对特定的IoT云。
- 在搜索深度的约束下，VerioT可以自动搜索模型中的所有可能状态。有一些委派系统可以具有数百个甚至数千个状态，很难手动检查。

