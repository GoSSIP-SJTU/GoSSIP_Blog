---
layout: post
title: "Wi Not Calling: Practical Privacy and Availability Attacks in Wi-Fi Calling"
date: 2019-09-24 05:14:58 -0400
comments: true
categories: 
---
作者：Jie Huang, Nataniel Borges, Sven Bugiel, Michael Backes

单位：CISPA Helmholtz Center for Information Security

出处：EuroS&P'19

原文：https://dl.acm.org/citation.cfm?id=3274753
<hr/>
## Abstract
Wi-Fi通话，也就是通过Wi-Fi提供的网络进行电话通信，目前已经得到了广泛的部署。然而由于Wi-Fi的固有性质，它的安全性很有可能比传统的LTE通话方式要弱。它使用了IETF所提供的IKEv2以及IPSec协议来保证通信过程的数据的可信与完整性。

在本篇文章中，作者首次对Wi-Fi电话的安全性进行了详细的分析。研究人员发现Wi-Fi电话协议中的漏洞可能导致用户信息泄露以及Dos攻击。通过搭建一个的伪造AP，攻击者可以获取手机与基站握手过程中泄露的IMSI (International Moblie Subsriber Identity)，尽管这种信息已经经过了加密。

<!--more-->

## 一、背景

传统的通话形式（比如打电话）所产生的数据流量是通过手机和基站之间直接传输来实现的，然后这些数据就在运营商的网络内部进行传送，而 WiFi 通话（WiFi Calling）则可以通过 WiFi 网络把通话的数据流量传送到运营商的内部网络，此时，并不需要通过基站就可以打电话、发短信等，并且在用户看来，用户感觉不到自己其实是通过 WiFi 来实现通话，用户的电话号码也像正常拨号一样可以被正常的显示。二者唯一的不同就是通话的数据信息从手机到运营商网络的路径不同，前者是手机直接和基站通信，后者是手机和 WiFi 进行通信。

按照目前情况来看，WiFi 通话在国内较少出现，而在国外则出现的比较多，因为这需要运营商的支持。WiFi 通话的作用也是比较明显的，比如，在高楼大夏里面，手机的网络信号一般都不是很好，有时候甚至只有一格或两格信号，而 WiFi 信号则不同，只要安装了路由器，就可以使用 WiFi，因此，在这种情况下，WiFi 电话是一个不错的选择，即使手机没有信号，只要能连接 WiFi，就能够正常的打电话、发短信等。

## 二、提出的方法以及解决的问题

由于 WiFi 电话需要通过 WiFi 来传输数据，相比于传统的数据传输方式（手机直接和基站进行数据传输），会出现更多的安全问题，比如，当手机连接到不可信的 WiFi 时，用户使用 WiFi 电话就有可能泄露个人隐私信息（SIM卡ID：IMSI （International Mobile Subscriber Identity）、用户所在的地理位置信息等），因此，作者提出了一套泄露用户隐私的测试方法，通过配置一套系统方案，获取用户的隐私数据，并在最后提出了一些针对性的缓解方法，防止用户隐私信息泄露，提高 WiFi 电话的安全性。

## 三、技术方法

### 1. WiFi 电话基本结构

WiFi 电话是基于 LTE（Long-Term Evolution） 体系结构而形成的，它的简化版结构如下图所示（图1）：

![图1 WiFi Calling Architecture](/images/2019-09-24/WiFi_Calling_Architecture.png)

图中有几个重要的名词：一个是 UE （User Equipment），一般是指手机；另一个是 ePDG（Evolved Packet Data Gateway），相当于数据包从公共网络进入运营商网络的一个网关；还有一个是 EPC（Evolved Packet Core），是一个基于 LTE 网络之上的一层核心网络框架，为用户提供语音和数据的传输。

传统情况下，手机打电话的时候，手机直接与基站通信，数据包可以直接进入运营商的网络（EPC），但是 WiFi 电话则使用普通的 WiFi 网络作为中间节点（图中的 AP ），然后再通过 ePDG 转发进入运营商的网络，ePDG 就是专门为了支持 WiFi 电话才引入 LTE 网络的一个中间设备，作为一个网关，控制公共网络和运营商网络之间的数据传输。

### 2. WiFi 电话连接的建立过程

当用户要发起 WiFi 电话的时候，UE 首先需要和 ePDG 建立一个连接，这篇文章就是利用这里的连接建立过程中产生的一些安全隐患来实现攻击，以及泄露隐私等。连接的建立过程如下图所示（图2）：

![图2 Wi-Fi Calling Handshaking Phases](/images/2019-09-24/HandshakePhase.png)

WiFi 电话通过两个 IETF（Internet Engineering Task Force）协议来保证数据传输的安全性：IKEv2（Internet Key Exchange）和 IPSec（IP Security）。连接的建立过程需要两个握手阶段，分别是：IKE Security Negotiation 和 IPsec Security Negotiation，IKE 用于实现双向认证（Mutual Authentication）和协商安全参数 SA（Security Associations），SA 就是包含一些列的策略（Policy）和密钥（Key）的一个数据结构，比如 UE 所支持的密码套件（AES256, SHA1 等）等，用于保护协商双方的信息。

上图中的（b）图就是把协商的第一阶段放大之后的过程，里面分为 4 个小步骤，图中的中括号括起来的信息表示是可选的信息，大括号括起来的信息表示加密的信息。四个小步骤如下所示：

（1）UE 给 ePDG 发送一个初始化请求信息（IKE_SA_INIT_REQ），该请求包含了与密码学相关的信息（比如密码算法 DH，Nonce 参数 Ni，还有一个 SA）。

（2）ePDG 检查从 UE 发过来的请求，并返回一个响应信息（IKE_SA_INIT_RES），该响应也包含了一些密码学相关的信息（比如密码算法 DH，Nonce 参数 Nr，还有 SA，以及一个可选的证书请求）。此时，ePDG 就已经可以计算出用于加密数据的共享密钥（SK_e）和用于认证或完整性检查的共享密钥（SK_a）。

（3）UE 收到响应信息之后也可以计算 SK_e 和 SK_a，此后，它们的数据通信就会被加密处理，UE 在这一步还会向 ePDG 发送一个认证请求信息（IKE_AUTH_REQ）并将自己的 IDi（IMSI） 放入其中，IMSI 会被加密保护。

（4）ePDG 收到认证请求之后，检查该请求信息并回送一个响应信息（IKE_AUTH_RES）。

### 3. 攻击场景

![图3 The Design of IMSI Privacy Attack](/images/2019-09-24/AttackDesign.png)

攻击场景如上图所示（图3），攻击者伪造一个无线热点（AP）和一个 IPSec 服务器，监听用户（UE）和真实服务器（ePDG）之间的数据通信，当用户发起握手请求时（图中的①），伪造 AP 可以截获并查看该请求的信息，并把该请求发送到真实的服务器中，当服务器发送响应信息的的时候（图中的②），伪造 AP 拦截该响应，并伪造一个响应给用户。但用户并不知道该响应信息被伪造了，因此，此时的用户是和伪造的 IPSec 服务器协商出共享密钥，就导致第三步的时候（图中的③），用户发送过来的认证信息被泄露。

![图4 Rogue AP Components and Attack Flows](/images/2019-09-24/AttackFlows.png)

作者在 Linux 系统上搭建了一个 WiFi 热点和一个伪造的 IPSec 服务器，如上图所示（图4），当用户（UE）连接到攻击者的 AP 之后，他的流量信息就会被攻击者监控，攻击者使用 Sniffer 模块来嗅探数据包的类型，当发现目标数据包类型（ISAKMP）的时候，开始发动攻击：提取用户 IKE_SA_INIT_REQ 请求中的信息 -> 伪造 IKE_SA_INIT_RES 请求 -> 获取 IKE_AUTH_REQ 中的 IMSI 并解密 -> 成功获取 IMSI。

攻击的前提是要首先监控到用户设备 UE 发起握手过程，可以通过监控 UE 发送 DNS 请求 ePDG 地址的过程（ePDG Discover），如下图所示（图5），图中④这个步骤就是请求 ePDG 服务器的地址，攻击者可以在伪造 AP 处获取该请求的返回数据，获取 ePDG 服务器的地址，进而继续监控 UE 发起通话时的握手过程，最后泄露用户的信息 （IMSI 等）。

![图5 DNS request](/images/2019-09-24/DNSRequest.png)

### 4. 应对策略



## 四、实验评估

### 1. 实验环境

- CPU： Intel I5
- 内存： -
- OS： Kali
- 工具：Wireshark、Scapy、Python、WiFi interface、iptables、hostapd、dnsmasq

### 2. 实验结果

**（1）IMSI 泄露攻击：**

作者测试了美国的四个厂商（T-Mobile、 Sprint、 AT&T、 Verizon），并从这四个厂商中抽取 10 个测试机器（Samsung Galaxy-Note-4/5、 Samsung Galaxy-5/6 和 iPhone 6/6s/7/8+）。测试结果如下表所示（表1）：

![表1 Test Results of IMSI privacy attacks](/images/2019-09-24/AttackResult.png)

从上表中可以看出，攻击结果不但泄露了用户的 IMSI 信息，还泄露了 ePDG 的地址，协商所使用的密码套件信息，以及证书相关的信息等。在这些测试厂商中，可以发现他们都没有使用证书，这也是由于在 IKE 标准文档中，证书被认为是非必须的，是可选的，因此导致用户信息被泄露。

**（2）DoS 攻击：**

这里的 DoS 攻击所攻击的对象就是 UE，也就是让 UE 打不了 WiFi 电话，或者切断他正在通话的过程。攻击场景如下图所示（图6），攻击分为两种情况：

第一种情况就是让 UE 打不通 WiFi 电话（图中的上半部分流程），攻击方法就是把 ePDG Discover 请求的返回值（Response）丢弃（Drop），或者篡改 Response 数据包中的 IP，让 UE 连不上 ePDG 网络。

第二种情况就是把正在通话的过程（概率性）切断（图中的下半部分流程），这种攻击方法很简单，只需要向 WiFi 网络中发送 Deauthentication Frame 即可。

攻击前提：

- UE 自动连接已知的 WiFi
- UE 在 WiFi 覆盖范围之内
- UE 开启 WiFi 电话模式（也就是默认通话使用 WiFi 电话，而不是 LTE 电话）

![图6 DoS attack](/images/2019-09-24/DoS_Attack.png)

作者对以下四台设备进行 DoS 攻击（第二种情况），攻击的结果如下表所示（表2），表中的 Call Drop 表示通话直接被中断了，而 Hand-off 表示通话被转移了（比如，从一个 AP 热点转移到另一个 AP 热点，或者是从 AP 热点转移到 LTE 网络中），这个转移策略是由 VCC（Voice Call Continuity） 特性决定的，VCC 的特性就是把一个通话 Session **无缝的**从一个网络转移到另一个网络。作者对每个手机进行 20 次测试，四个手机平均的通话中断率是 26.25%。此外，当前的实验环境还包含：（a）LTE 网络信号很好，（b）当前存在多个已知的 WiFi（UE 可以自动连接的 WiFi）。

![表2 DoS Attack Results](/images/2019-09-24/DoS_AttackResult.png)

**（3）缓解措施：**

- 用户关闭手机的自动连接 WiFi 功能，不要随意连接陌生的 WiFi，这是最有效的缓解方法。
- 在 IKE 认证过程中增加 PKI（public key infrastructure）认证体系（例如数字签名）。
- 设置一个隐藏种子（Hidden seed value），用于辅助密钥交换过程（验证 ePDG 的合法性），比如使用该种子配合生成一个哈希（hash）或者是随机数（nonce）；但是这个种子要唯一，并且不能被第三方知道（但是我觉得这个方法不行，或者是我没有完全理解他的这个方法）。
- Hand-off 不应该由 Deauthentication Frame 决定，应该由信号的强度决定，这样可以避免第二种 DoS 攻击。
- 厂商（比如移动网络厂商）在目标区域安装无线入侵检测系统 WIDS（Wireless Intrusion Prevention System）（感觉这个方法不可行）。
- 用户手机安装检测软件，用于检测伪造的 AP（感觉这个方法就是胡扯）。

## 五、优缺点

### **优点：**

- 根据泄露的 IMSI 可以查询到用户的电话号码，以及用户所在的地理位置，这两个信息相对比较有用，也是人们比较关心的两个隐私数据。
- 攻击简单、有效，在公共 WiFi 网络中（比如酒店 WiFi 等）更容易实施。

### **缺点：**

- 稍微有点安全意识的人都不会随便连接到免费的、陌生的 WiFi 网络中，因此，这个攻击的前提条件就比较难满足。
- 对用户打电话进行攻击没啥特殊的意义，最多就打不通电话罢了，或者原来连着 WiFi 打不通 WiFi 电话，就换成打普通电话就好了。

## 六、总结和看法

这篇文章的切入点就是利用 WiFi 电话在建立连接的时候缺乏双向认证（Mutual Authentication）步骤，从而导致了用户设备的隐私信息被泄露（例如 IMSI 等），同时，利用这个弱点，还可以对用户设备实施 DoS 攻击，使得用户设备无法连接 ePDG，无法拨打 WiFi 电话，或者使得正在拨打的 WiFi 电话断线等。而对应的应对方式，其实只要用户不要随意的去连接陌生 WiFi 就可以了。个人感觉文章内容相对较少，就是设置一个恶意的 WiFi 让用户连接，并伪造一个 ePDG 来获取用户的 IMSI 信息；实验相对简单，攻击场景也相对受限，应该很少人会连着一个陌生的 WiFi 来打 WiFi 电话，而且如果用户连接了恶意的 WiFi，可能导致的后果比 DoS 攻击和泄露 IMSI 更加严重（比如：各种登录的信息可能就会被泄露）。




