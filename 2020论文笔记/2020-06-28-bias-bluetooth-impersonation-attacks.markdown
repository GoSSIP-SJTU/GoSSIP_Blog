---
layout: post
title: "BIAS: Bluetooth Impersonation AttackS"
date: 2020-06-28 21:07:53 +0800
comments: true
categories: 
---

> 会议： IEEE Symposium on Security and Privacy 2020 
>
> 作者： Daniele Antonioli，Nils Ole Tippenhauer， Kasper Rasmussen
>
> 单位： School of Computer and Communication Sciences EPFL
>
> ​			 CISPA Helmholtz Center for Information Security 
>
> ​			 Department of Computer Science University of Oxford 
>
> 链接： [BIAS: Bluetooth Impersonation AttackS](https://francozappa.github.io/about-bias/publication/antonioli-20-bias/antonioli-20-bias.pdf)

### 1 Introduction

蓝牙通信标准包含LSC（传统安全连接）和SC（安全连接）两种连接机制。

在本文中，作者展示了蓝牙规范中存在一些漏洞，使得攻击者能够在安全连接建立阶段实施冒充攻击。这些漏洞包括缺少强制性双向认证、角色转换的过度授权以及身份认证过程降级。作者详细的描述了每一种漏洞，并且利用他们来实现一系列攻击。作者把此类攻击称为Bluetooth Impersonation AttackS (BIAS)。

由于作者的攻击是针对蓝牙通信标准，因此对于任何符合标准的蓝牙设备，不管其使用的蓝牙版本、安全模式、设备制造商以及实现细节，攻击都是有效的。此外，由于蓝牙标准不要求通知终端用户认证过程的结果或者缺乏双向认证，因此这种攻击是隐蔽的。

通过此类攻击，作者成功的攻击了31台蓝牙设备。

本文主要贡献如下：

（1）作者描述了BIAS攻击模型，利用蓝牙标准中的缺陷，在不知道共享长期密钥的情况下，建立安全连接。

（2）作者提供了BIAS工具包，能够自动对蓝牙设备进行BIAS攻击。

（3）作者成功的攻击了16个传统连接LSC设备以及15个安全连接SC设备，评估28个蓝牙芯片，以此说明攻击的严重性。

<!-- more -->

### 2  SYSTEM AND ATTACKER MODEL 

#### 2.1 System Model

作者考虑两台受害者设备，Alice和Bob，两者通过一个安全蓝牙连接来进行通信，如图1所示。攻击条件并不要求两台设备在攻击期间通信，仅需要两台设备存在并且在过去进行过通信即可。

![](/images/2020-06-28/fig1.PNG)

作者假设Alice和Bob已经共享一个长期密钥，作为link key（KL）。这个密钥已经被蓝牙SSP协议和强关联模型协商过。

作者假设Bob是蓝牙主节点，Alice是蓝牙从节点。Bob想要使用已有的link key和Alice建立一个安全连接；Alice想要使用这个key接收一个来自Bob的连接。

冒充攻击是在安全连接建立的过程中发生的。作者假设这个过程中所有的安全原语都是perfectly secure的，Alice和Bob之前已经使用长期密钥建立过连接。

#### 2.2 Attacker Model

攻击者（Charlie）的目标是冒充Bob（或Alice），和Alice（或Bob）建立一个安全的蓝牙连接。

- C不知道A和B之间共享的长期密钥
- C能够窃听、解码和控制未加密的数据包，以及干扰蓝牙频谱
- C知道Alice和Bob的公开信息，包括蓝牙名称、蓝牙地址、协议版本号和功能

由于安全连接建立过程没有加密，C可以通过窃听A和B的通信来收集它们的特征；在A和B之间的安全连接建立完成之后，C可以干扰蓝牙频谱来使得A和B断连，并重新建立安全连接。

### 3  BLUETOOTH IMPERSONATION ATTACKS (BIAS) 

A和B在一次配对后共享KL，这之后再次利用LSC或SC建立连接时，验证是否拥有KL，安全标识符为一个三元组（KL，BTADDA，BTADDB）。当冒充A或B时，C可以将其蓝牙地址更改为BTADDA或BTADDB，但是不能证明他拥有KL。

本文中蓝牙假冒攻击场景：

1）蓝牙安全连接建立没有加密或完整性保护

2）传统安全连接LSC的建立连接阶段不要求双向认证

3）蓝牙设备可以在任意时间执行角色转换

4）设备使用安全连接SC配对的也能够使用传统安全连接LSC建立连接

#### 3.1  BIAS Attacks on Legacy Secure Connections 

A和B想要利用LSC建立一个连接，通过验证KL来进行身份验证，过程如图10。

- 主节点计算并发送CM到从节点，从节点计算响应RS = HL(KL, CM, BTADDS)并把它发送给主节点
- 主节点计算响应RS是否正确，如果正确，则认为它与从节点具有相同的KL。

![](/images/2020-06-28/fig10.PNG)

C冒充主节点B，攻击流程如图2：

- C利用B的公开信息冒充B，请求与A建立连接
- A接受连接请求
- C发送CM给A
- A计算RS，并将其发送给C
- C冒充B进行会话密钥协商和安全连接，而无需证明他拥有KL

![](/images/2020-06-28/fig2.PNG)

C也可以冒充从节点，然后通过蓝牙的角色转换过程执行攻击。

蓝牙标准规定，在baseband paging完成后，可以随机切换主从角色。这是存在问题的：C可以模拟从设备，并在单向身份验证之前启动角色切换，成为主设备，从而不需要进行身份验证。

攻击流程如图3。

![](/images/2020-06-28/fig3.PNG)

#### 3.2  BIAS Downgrade Attacks on Secure Connections 

安全连接SC提供一个双向认证过程，如图11所示。

- A（从节点）和B（主节点）之间交换CS和CM，没有特定的顺序
- A和B计算RM || RS = HS(KL, BTADDM, BTADDS, CM, CS)

- 由于蓝牙标准没有明确规定响应顺序，在实验中，通常是从节点先发送RS
- A发送RS到B，B发送RM到A
- A和B验证响应是否匹配，如果双方验证均通过，则KL被双向验证

![](/images/2020-06-28/fig11.PNG)

BIAS安全连接降级攻击在安全连接场景中实施。

由于蓝牙协议并没有规定使用SC进行配对的两台设备必须使用SC来建立安全连接，因此A和B使用SC配对后，仍可以使用LSC进行安全连接建立。

假设A和B已经配对并且支持SC，C伪装成B，攻击流程如图4所示。

![](/images/2020-06-28/fig4.PNG)

- 在特征交换阶段，C冒充B告知A它不支持SC，那么SC就会降级成LSC
- 然后C就能够与A进行连接，而不需要认证

C也可以冒充A，攻击流程如图5所示。

![](/images/2020-06-28/fig5.PNG)

### 4   BIAS REFLECTION ATTACKS ON SECURE CONNECTIONS 

作者描述了使用另一种方式——反射攻击来攻击安全连接SC认证。

在反射攻击中，攻击者欺骗受害者回复他自己的挑战，并且发送响应给攻击者；然后攻击者就可以重放该响应给受害者来通过身份验证。

反射攻击假设在接受到来自远程受害者的响应之后，C能够安全认证过程中进行角色转换。蓝牙标准并没有明确对于在认证过程中间实施角色转换进行规定。作者假设从节点先发送R。

C冒充B（主节点），攻击流程如图6所示。

![](/images/2020-06-28/fig6.PNG)

- C冒充B，向A发送连接请求，A接受连接请求
- C发送CM到A，A发送CS到B；由于C不知道KL，因此他无法计算RM
- 在A发送RS响应给C之后，C发送一个角色转换给A
- A接受角色转化，C成为新的从节点；A成为新的主节点，并想要来自C的RS
- C将RS转发给A，完成安全（双向）认证过程。

C同样可以冒充A，攻击流程如图7所示。

![](/images/2020-06-28/fig7.PNG)

### 5  IMPLEMENTATION 

作者实现了上述BIAS攻击。

实验环境为：CYW920819EVB-02 评估板和Linux笔记本电脑。

#### 5.1 BIAS Toolkit

在通过对开发板固件逆向收集完足够多的信息后，作者开发了BIAS工具包来自动化的实现BIAS攻击。BIAS工具包是第一个实现蓝牙冒充攻击，开源代码 https://github.com/francozappa/bias。在实验中，作者使用该攻击包成功的攻击了31台蓝牙设备（28台不同的蓝牙芯片）。

![](/images/2020-06-28/table9.PNG)

图9描述BIAS工具包，该工具包将Impersonation File (IF)和Attack File (AF)作为输入。

- IF文件包含所要冒充设备的信息，例如蓝牙地址、蓝牙名称、Secure Connections支持。
- AF文件包含攻击设备的信息，例如笔记本电脑使用的HCI接口的名称、想要在开发板蓝牙固件中patch的函数地址。

### 6 EVALUATION

#### 6.1 Evaluation Setup

攻击场景：

- A和攻击设备是两块开发板，支持SC；B是其他蓝牙设备，可能支持SC。
- A与B配对，攻击者不知道KL。
- 攻击者冒充A，尝试与B建立安全连接。

作者执行4种BIAS攻击。

1）LSC MI: LSC 主节点冒充

2）LSC SI: LSC 从节点冒充

3）SC MI: SC 主节点冒充

4）SC SI: SC 从节点冒充

本文的实验评估设置能够实现在几分钟内测试BIAS攻击，并且成本很低。攻击设备包含连接到Linux笔记本电脑的CYW920819EVB-02开发板。

#### 6.2 Evaluation Results

表3显示了评估结果。第一列为蓝牙芯片名称，第二列为评估使用芯片的设备名称，第三列和第四列为LSC MI、LSC SI BIAS攻击的评估结果，第五行和第六行为评估SC MI和SC SI BIAS攻击的结果。实心圈表示芯片和相关服务易受到攻击，空心圈表示芯片和相关服务不受攻击的影响。由于安全连接SC在蓝牙标准中是可选的，作者使用-表示芯片或设备不支持安全连接。

![](/images/2020-06-28/table3.PNG)

表3证实了所有31台蓝牙设备都会受到BIAS攻击的威胁。总的来说，作者攻击了16台LSC设备和15台SC设备，包含来自Intel、CSR等的蓝牙芯片以及来自Android、Apple、Linux、Microsoft、Cypress和CSR的设备，覆盖了蓝牙5.0、4.2、4.1以及4.0及以下版本。

有一个例外是ThinkPad 41U5005鼠标，这个鼠标不受LSC SI攻击。当作者让鼠标与攻击设备建立一个安全连接时，即使攻击设备切换角色并且完成单向身份认证，鼠标总是会要求攻击设备在开始会话密钥协商之前进行认证。

此外，表格证实了BIAS攻击是针对标准的，因为攻击并不针对特定的蓝牙芯片、蓝牙主机栈、安全连接的使用和蓝牙版本号。

### 7 DISCUSSION

这一部分作者讨论了如何将BIAS攻击和KNOB攻击结合在一起，同时也讨论了BIAS攻击产生的根本原因和相应对策。

#### 7.1 BIAS和KNOB攻击结合

BIAS攻击和KNOB攻击都是针对标准的，并且它们是利用蓝牙安全连接建立的不同阶段。

- BIAS针对连接密钥认证，允许攻击者在无需连接密钥的情况下作为主从节点认证

- KNOB攻击针对会话密钥协商，允许攻击者降低会话密钥的熵值，来进行暴力破解。 攻击设备可能会干扰用于在两个设备之间的 BR/EDR 连接上设置加密的过程，从而缩短所使用的加密密钥的长度。 

  只有KNOB攻击不能冒充一台蓝牙设备，因为攻击者不能处理长期密钥。

BIAS和KNOB攻击可以结合在一起来实现冒充一台蓝牙设备、在没有连接密钥的情况下完成认证、协商会话密钥、建立安全连接以及暴力破解会话密钥。

#### 7.2 BIAS Attacks Root Causes

1）Integrity：蓝牙安全连接建立没有完整性保护和加密保护。

2）Legacy Mutual Authentication：蓝牙LSC没有要求使用双向认证过程。攻击者可以冒充verifier，并在不经过身份验证的情况下完成安全连接建立。

3）Role Switching：蓝牙角色切换可以在任意时间执行，攻击者可能会作为从节点开启安全连接建立过程，然后变成主节点来进行认证。

4）Secure Connections Downgrade：蓝牙不会在配对或连接建立阶段要求安全连接，两台能够使用SC进行配对的设备也能够使用LSC进行配对。攻击者利用这个漏洞可以来降级一个SC为LSC，以使用有问题的单向认证过程。

![](/images/2020-06-28/table4.PNG)

#### 7.3 BIAS攻击缓解策略

1）Integrity：强制使用长期密钥KL来保护安全连接建立。

2）Legacy Mutual Authentication and Role Switching：使用双向身份认证过程。攻击者必须要有认证长期密钥KL。

3）Secure Connections Downgrade：强制两台使用安全连接SC配对的设备在安全连接建立阶段总是使用安全连接SC，或者需要用户来决定是否同意安全连接降级。