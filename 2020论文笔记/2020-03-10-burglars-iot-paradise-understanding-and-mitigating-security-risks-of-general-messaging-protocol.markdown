---
layout: post
title: "Burglars' IoT Paradise Understanding and Mitigating Security Risks of General Messaging Protocol"
date: 2020-3-10 04:21:03 -0400
comments: true
categories: 
---

> 作者: Yan Jia, Luyi Xing, Yuhang Mao, Dongfang Zhao, XiaoFeng Wang, Shangru Zhao, and Yuqing Zhang
>
> 单位: School of Cyber Engineering, Xidian University, China; National Computer Network Intrusion Protection Center, University of Chinese Academy of Sciences, China; Indiana University Bloomington, USA
>
> 会议: Oakland2020
>
> 链接: http://homes.sice.indiana.edu/luyixing/bib/oakland20-mqtt.pdf

## 1 概述
本文首次对主要的IoT云平台与设备通信消息协议的安全性进行了系统研究，作者发现这些云平台上的消息传输协议很容易受到攻击，并且利用这些漏洞进行攻击会造成严重后果。为此，作者提出了IoT云端通用消息协议的安全原则，并且提出和实施了端到端保护的方法。

<!--more-->

## 2 背景知识

### 2.1 Cloud-based IoT Communication

一个基于云端IoT系统通常包含三部分：cloud（本文中指云平台）、IoT Device、user's management console（手机应用）。

如图1 所描述的，云是这类系统的关键部分，它控制着设备和应用之间的通信，并且需要对设备和应用进行配对和身份认证。

![图1 cloud-based IoT communication架构](/images/2020-03-10/fig1.PNG)

- MQTT

  MQTT是IoT中常用的发布-订阅（publish-subscribe）消息传递协议，它是一个应用层轻量级协议，能够在资源受限的设备上运行。
  MQTT通信协议的核心是MQTT消息代理（message broker），如图1所示。

  以代理为连接枢纽，MQTT使用发布-订阅模式进行通信：
    - MQTT客户端（设备或应用）将消息发布到代理托管的特定主题（topic）
    - 代理将消息路由到另一个订阅了该主题的客户端

  MQTT客户端向代理发送三类基本类型消息：CONNECT（建立连接）、PUBLISH（发布主题消息）、SUBSCRIBE（接收订阅消息），如图2所示。
  ![图2 MQTT通信](/images/2020-03-10/fig2.PNG)

### 2.2 MQTT保护机制

MQTT本身缺乏身份认证和授权机制，在实践中不同厂商IoT云平台通常会实施自定义安全机制，本文主要关注MQTT的认证和授权机制。

- Client Authentication: MQTT可以通过不同的方式进行身份验证，如用户名/密码、证书等。
- Client Authorization: 授权机制保证每个用户只能与允许的设备进行通信。

### 2.3 威胁模型

- 攻击者能够开通IoT平台账户，能够收集和分析平台、设备与自己应用之间的网络流量，但不能窃听其他用户设备和应用之间的流量。
- 设备共享。攻击者可能具有设备临时访问权限。例如度假酒店为访客授予临时访问设备的权限。

## 3 MQTT IoT 通信的安全分析

作者将MQTT通信分为4个实体：标识符、消息、主题和会话（session），分别研究分析MQTT协议的安全性。下面描述作者在对MQTT协议分析过程中所发现的问题。

### 3.1 Unauthorized MQTT Messages

场景：用户仅当具备临时访问设备权限的情况下才会被信任，而不能在之前或之后获得任何信息或干扰设备运行。

#### Unauthorized Will Message

Will消息由客户端提前设定，可以包含控制命令或文本。一旦客户端意外断连（未发送DISCONNECT消息），代理会发布Will消息到所有订阅该主题的客户端，来使他们采取相应的操作。

在设备访问权限转移的情况下，这种意外处理机制会存在问题。攻击者（前用户）可以注册一条Will消息，在他不具有访问权限时触发该消息，来实现攻击。

作者使用iRobot Roomba 690在AWS IoT云端实现了利用Will消息的PoC，如图3所示。IBM、Baidu、Tuya Smart等云平台存在与AWS相同的问题。

![图3 iRobot设备上的遗嘱消息攻击](/images/2020-03-10/fig3.PNG)

#### Unauthorized Retained Message

保留消息（retained message）是指当MQTT客户端发布了一条消息到相应的主题，而没有客户端订阅该主题（例如设备离线），代理会存储每个主题的最后一条消息（保留消息），当订阅该主题的客户端上线时，代理会将该保留消息投递给它。

攻击者（前用户）在具有设备访问权限期间可以向设备订阅的主题发布保留消息，然后在失去权限时等待设备上线并接收保留指令，以实施攻击。

作者发现Baidu IoT Cloud和Eclipse Mosquitto（开源的MQTT代理）会受到此类攻击。

### 3.2 Faults in Managing MQTT Sessions

场景：MQTT通信是通过在客户端和代理之间建立会话进行的，会话与MQTT客户端状态关联，因此当客户端状态发生变化（如撤销授权），其已建立的会话状态也要更新。

#### Non-updated session subscription state

场景：如果用户被撤销所有权限，在任何已建立的会话中，该用户将不被允许进行任何操作。

然而，作者发现在MQTT协议规范中没有任何关于在客户端状态变化时更新会话状态的规定和说明。这会导致如果攻击者（前用户）建立了订阅主题的会话，即使之后撤销他的权限使其不能够订阅主题，代理仍然会通过已建立的会话向攻击者（前用户）传递消息，使得攻击者可以接收到当前用户的消息。

作者确认了很多IoT平台（IBM、Tuya、Alibaba、Baidu）上存在这种不安全的会话状态管理。

#### Non-updated session lifecycle state

MQTT客户端可以是设备或用户，通常情况下，IoT云平台将设备看作要访问的资源，将用户看作需要进行身份验证和授权的主体。作者发现这种差异存在安全隐患，当设备由新用户重置时，会删除前用户对设备的访问权限，但是不能撤销设备访问MQTT代理主题的权限。

如果攻击者（前用户）在权限期内获得设备身份凭据（流量分析或逆向工程），那么即使新用户重置设备，攻击者也可以利用设备凭据来伪装成设备发布虚假消息到相应主题。一些IoT平台采取了措施来防止这种攻击，当用户更改时，设备会被强制过期。

但是作者发现如果攻击者在凭据过期之前建立会话并保持会话在线，那么即使凭据过期，他仍可以通过该会话伪造设备消息。这是由于平台在重置设备时，没有更新已建立MQTT会话状态。

作者在IBM、Tuya、Alibaba、Baidu等平台中确认了该问题。

### 3.3 Unauthenticated MQTT Identities

一个IoT平台账户可以有多台设备，每个设备有自己的身份标识ClientId，一台设备可以在多个账户之间共享。作者指出如果这种关系不能安全管理，会使MQTT通信受到攻击。

#### ClientId hijacking

MQTT协议要求代理在发现相同ClientId的新客户端时，断开与原客户端的连接，这会造成DDoS攻击。作者还发现在这种情况下，MQTT代理和客户端能够重新加载之前的会话，恢复之前的状态。

攻击者可以拥有合法的平台账户，即能够建立起与平台的MQTT连接，如果平台不能正确处理平台账户与设备身份的安全访问机制，攻击者可以利用上述机制，通过已知的ClientId来恢复会话并窃取信息。例如，攻击者利用自己的平台身份和通过某种手段获取的受害设备的ClientId，在平台建立MQTT连接，那么即使攻击者没有订阅该设备的主题，仍然可以收到该设备的消息。

攻击者可以通过两种方式来获取ClientId：

- 猜测：ClientId是一个具有特定语义、格式的序列，由于MQTT规范仅要求ClientId的唯一性，大多数IoT云平台建议使用便于管理的ClientId，例如MAC地址、递增分配序列号。攻击者可以从一个已知的ClientId开始进行穷举搜索。
- 共享设备：如果攻击者可以短期访问该设备，那么该设备的ClientId会永远暴漏给攻击者。

作者使用iRobot Roomba 690在AWS IoT云平台上实施了攻击。通过搜索设备该设备令牌附近的序列，来推测了一组可能的iRobot产品设备ClientId，结果显示这样能够造成大规模的DDoS攻击。

作者在AWS、Baidu IoT Cloud、IBM等平台上发现了此类问题。

### 3.4 Authorization Mystery of MQTT Topics

MQTT主题结构类似分层文件路径，例如/doorlock/[device ID]/status。作者发现IoT云平台上如果存在对MQTT主题的不正确描述，可能会导致授予攻击者过多的访问权限。

#### Insecure shortcut in protecting MQTT topics

由于IoT平台可能需要管理成千上万的设备和用户，作者发现很多IoT云平台采用了快捷方式来进行授权。

苏宁的IoT云服务隐含地假设MQTT主题是保密的，因此用户可以订阅所知道的任何MQTT主题。但是这种假设在实际中根本不成立，如果攻击者可以短期内使用目标设备，那么他可以轻松了解该设备的主题（流量分析）。此外，一些厂商将设备唯一标识符用于其MQTT主题，如设备序列号或MAC，可以受到穷举攻击，这样攻击者仍然可以订阅该设备的主题，窃取隐私信息。

作者使用HONYAR Smart Plug IHC8340AL实施攻击，通过对苏宁Smart Living手机应用和云平台之间的流量进行分析，可以找到它的MQTT主题。作者使用脚本成功订阅了该设备的所有主题，之后即使重置设备使用另一个账户，同样可以进行攻击。

#### Expressive syntax of MQTT

一台设备可能有很多关联的主题（例如/deviceID/cmd用于传递命令，/deviceID/status用于更新状态），为了便于使用，云平台通常允许用户使用通配符#或+来订阅该设备甚至多个设备的多个主题，这存在很大的安全隐患。

AWS上，即使存在禁止用户访问一个类似deviceId/cmd的主题，如果攻击者能够订阅deviceId/#，该主题表示代理上deviceId下的所有MQTT主题，那么攻击者仍然可以访问那些受保护的主题（deviceId/cmd），使得策略被绕过。苏宁云平台上存在类似更严重的问题，任何用户都可以订阅#主题，该主题表示代理上的所有MQTT主题。

作者进行了实验，通过一个脚本与苏宁Smart Living云平台建立了MQTT通信并通过了验证，成功订阅#主题（平台通用主题）。通过订阅，作者收到了来自智能锁、摄像头、监控等大量隐私信息，甚至能够以此来推断家庭/同居关系、行为习惯等。此外，泄露的信息还包括云下所有设备的ClientId，攻击者能够进行DoS攻击使任意用户设备脱机。

## 4 MEASUREMENT

作者对当前流行的IoT云平台的MQTT通信进行了评估，结果如表1。

![表1 评估结果总结](/images/2020-03-10/table1.PNG)

其中对于message authorization，除了不支持两类消息（Will和Retained）的平台，所有平台都存在问题。

此外，作者还进行了实验来评估这些问题带来的影响。

通过订阅苏宁代理的通用#主题，攻击者可以收集到云下所有设备的消息。作者收集到三周内一共8亿条真实的MQTT消息，包含了大量的设备信息（设备ID、类型、状态、位置、收集的信息等）和用户信息（PII）。作者将这些信息组合起来纵向分析，能够推断出用户的私人习惯、行为或关系。

例如表4显示了门锁三周内的状态：用户通常工作日在家，周五出门。攻击者还可以通过类似“[Person Name set by user] opened the door via fingerprint”来推测用户的私人关系等。

![图4 苏宁云平台上一个智能门锁的"Open"状态](/images/2020-03-10/fig4.PNG)

## 5 MITIGATION

本节作者介绍了一种用于保护协议实体的设计原则以及一种增强的访问控制模型，作者在Mosquitto上实现该方案并且对其有效性进行了评估。

### 5.1 Managing Protocol Identities and Sessions

研究表明IoT云平台通常通过平台账户来认证MQTT连接，只要用户拥有平台的合法账户，就能够使用任意ClientId建立起与平台的MQTT连接。保护措施如下：

- 使ClientId以用户的平台标识p_user_id开头。但对硬编码ClientId无效。
- 维护平台标识与允许的ClientId的映射，任何未授权的ClientId都会被拒绝。

此外，消息协议中的会话状态也应随着用户状态的变化而更新。

### 5.2 Message-Oriented Access Control Model

作者提出了基于UCON的IoT增强通信访问模型——Message-Oriented Usage Control Model (MOUCON)。它将消息当作资源（Object），然后根据属性检查访问权限，相关定义如下。

- Subject (S)：通信中客户端的集合，包括用户和设备。一个subject由它的属性来定义描述。
- Subject Attributes (ATT(S))：S的属性可以描述为 ATT(S) = {id, URIw, URIr}，id为标识信息，URIw为允许发送消息的URI集合，URIr为允许接收消息的URI集合。
- Object (O)：S拥有权限的消息集合。
- Object Attributes (ATT(O))：对象的属性被描述为 ATT(O) = {content, URI, source}，content是应用层信息，URI表示消息信道，source标识对象的来源。
- Right (R)：Right是指一个主题拥有和对一个对象进行操作的权限，有两种通用类型：Read (R)和Write (R)。
- Authorizations：授权函数通过授权规则来评估ATT(S)和ATT(O)以及要求的权限，决定是否能进行访问。

示例：allowed(s, o, R) ⇒ (o.URI ∈ s.URIr) ∧ (o.URI ∈ o.source.URIw)

其中，allowed(s, o, R)指的是客户端S被允许使用权限R来对对象O进行操作，这既需要检查客户端S（接收方）是否具备对对象O进行读的权限，又要检查发送方（o.source）是否具备对对象o进行写的权限。

### 5.3 Implementation and Evaluation

作者在Mosquitto 1.5.4上实现了上述保护，修改了相关的数据结构来添加安全相关属性，并将授权功能添加到代理。

作者通过将实施保护的安全Mosquitto和原版本Mosquitto对比来对方案进行评估。作者实施了之前提到的攻击，结果表明安全的Mosquitto击败了所有攻击，而另一个未捕获任何攻击。

为了评估性能开销，作者记录了消息发布和接受之间的平均延迟，以及该时段内的平均CPU和内存使用情况，如表2所示。结果表明方案消息传输延迟（最多0.63%）和内存开销（最多0.16%）与提供的保护能力来比微不足道。

![表2 性能评估](/images/2020-03-10/table2.PNG)

