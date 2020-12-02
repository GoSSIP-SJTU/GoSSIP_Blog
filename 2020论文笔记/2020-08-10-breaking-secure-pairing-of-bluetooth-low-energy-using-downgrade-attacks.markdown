---
layout: post
title: "Breaking Secure Pairing of Bluetooth Low Energy Using Downgrade Attacks"
date: 2020-08-10 11:32:28 +0800
comments: true
categories: 
---

> 作者： Yue Zhang, Jian Weng, Rajib Deyγ, Yier Jin, Zhiqiang Lin, and Xinwen Fu 
>
> 单位：
>
>  College of Information Science and Technology, Jinan University Department of Computer Science, University of Central Florida Department of Electrical and Computer Engineering, University of Florida Department of Computer Science and Engineering, Ohio State University 
>
> 会议：Usenix Security '20
>
> 链接： http://jin.ece.ufl.edu/papers/USENIX2020-BLE.PDF 

### 1  Introduction

BLE配对方法包括四种：1）Just Works；2）Passkey Entry；3）Numeric Comparison；4）Out Of Band (OOB)。其中，Passkey Entry和Numeric Comparison是最常使用的两种安全配对方法。

为了防止MITM攻击，最新的BLE 4.2和5.x添加了一个新的SCO模式，在这种模式下，强制BLE设备使用安全连接方法来认证手机/用户设备，例如Passkey Entry和Numeric Comparison。然而，BLE标准并没有要求发起者（手机设备）使用SCO模式，也没有定义BLE编程框架应该怎么实现这种模式。

在这篇论文中，作者指出BLE发起者的编程框架必须要正确处理SCO初始化、状态管理、错误处理以及绑定管理，否则一些设计缺陷能够被利用来实现降级攻击，从而强制BLE配对协议在一个不安全的模式下运行，但用户并不会发现。

本文主要贡献如下：

- 本文首次发现了在SCO模式中BLE编程框架存在的设计缺陷。
- 作者使用18台BLE设备和以及5台Android手机实施了攻击来证实Android BLE编程框架中存在这些缺陷。
- 作者提出并实现了相应的对策，并对这些缓解措施进行了测试与评估。
- 扩展实验证实了这些设计缺陷也存在于所有主流的操作系统，包括iOS、macOS、Windows和Linux。

<!-- more -->

### 2 BLE Workflow

如图2所示，一个典型的BLE主节点（例如，手机等移动设备）与从节点（例如，血压监测仪等BLE设备）的流程包含11步，大致分为了Connection、Pairing、Communication三个阶段。

![](/images/2020-08-10/fig2.PNG)

在配对阶段，移动设备和BLE设备主要进行如下操作：

**Phase 1 - Pairing feature exchange：** 两台设备交换配对特性来协商配对方法

- Authentication requements：包括Bonding和MITM保护
  - Bonding指配对阶段生成的密钥会保存留作之后使用，以减少后续配对的延迟.
  - MITM保护对MITM攻击的抵御，如果该值被设置为false，双方会使用Just Works进行配对；如果有一方设置为true，则会有两种情况：如果双方设备都支持，则选择Passkey Entry或Numeric Comparison方法；否则选择Just Works方法。
- I/O capabilities：交换设备支持I/O能力，决定使用什么配对方法。
- BLE version：在Secure Connection (SC) 位上设置，如果设置，则采用BLE 4.2及以上；否则使用BLE传统配对协议。

**Phase 2 - Key exchange and authentication：**

- Public key exchange：使用ECDH密钥交换协议，生成一个对称密钥DHKey
- Authentication stage 1：根据Phase 1的认证要求，选择配对方法
- Authentication stage 2 and LTK calculation：根据DHKey生成MacKey和LTK，MacKey用于保证设备生成相同的LTK

**Phase 3 - Transport specific key distribution：**

双方设备由LTK生成SessionKey来加密通信，其中Identity Resolution Key (IRK)可能会有一台设备发送到另一台，以用于隐私保护。

例如如果一台移动设备需要保护其MAC地址，那么它会先将它的IRK和MAC地址分发给配对设备，然后移动设备会使用IRK生成一个可解析的私有地址，配对设备可以使用IRK来解析私有地址。

### 3 SCO Mode Design Flaws

作者介绍了四种支持SCO模式的关键能力，并指出由于缺少这些能力所导致的设计缺陷。

#### 3.1 Specification Deficiency

对于一个从设备，BLE规范定义了SCO模式。

- 在该模式下只能使用三种安配对方法：Passkey Entry、Numeric Comparison和Secure OOB
- 不允许使用BLE Legacy（不能使用传统身份认证）。
- 如果不使用安全方法配对，则设备应发送配对失败的数据包，错误代码为"Authentication Requirements"。

BLE规范为提供服务的从节点指定了SCO模式，但它并没有明确定义主节点（如移动设备）的SCO模式。这样情况下，攻击者可以欺骗受害者（例如，使用伪造的血压计）并连接到受害者设备以发起各种攻击。

作者发现以下四个阶段对于在主节点实施SCO模式至关重要：启动、状态管理、错误处理和绑定管理。由此，作者指出主节点（发起方）应具备以下四种能力：

- Initiation: 一个手机应用应该能够控制操作系统来使用安全配对方法。
- Status management: OS应该记住指定的安全配对方法，在适当的时间通知应用。适当的时间应该是在图2中的Step 5和Step 6之间。
- Error handling：在通信过程中发生错误时，OS和应用应该一起处理这些错误，并执行指定的安全配对方法。
- Bond management：应用应该能够移除由于错误损坏的绑定以再次启动执行过程。

表1列举了一个操作系统可能有的四种设计缺陷。

![](/images/2020-08-10/table1.PNG)

#### 3.2 Design Flaws in Android

作者详细描述了上述四种BLE规范设计缺陷在Android中的具体表现。

**（1）Flaw 1 - 没有能够指定安全配对方法的机制**

createBond()函数是Android app中唯一能够开启配对过程的函数，它不接受任何输入参数，app不能指定任何特定的配对方法。该函数的返回值为true或false，表示配对过程是否成功开启。

createBond()也检查手机设备中是否存在LTK，如果存在，则该函数返回flase，并且将不会重新配对，因为手机设备已经和设备配对过。

![](/images/2020-08-10/listing1.PNG)

**（2）Flaw 2 - 没有能够及时执行指定安全配对方法的机制，app不能及时获知协商的配对方法**

Android仅依赖交换的I/O特性来确定配对方法，app可以使用以下异步机制来在配对完成后获取一个配对过程的状态，包括所协商的配对方法。

- Intent ACTION_BOND_STATE_CHANGED
- Intent ACTION_PAIRING_REQUEST

因此，app不能及时执行一个指定的安全配对方法。

**（3）Flaw 3 - 如果BLE stack错误的处理了配对错误，app无法对此进行处理**

Android BLE框架没有为app提供API来正确的处理配对错误。

配对错误包含以下几种：

- Pin or Key Missing (0x06)：配对完成后，如果BLE设备的LTE被移除，设备会发送该错误码。然而Android手机并不会通知用户，而是自动以明文的方式进行通信，并且没有API和机制可以让app检测到0x06错误。
- Insufficient Authentication (0x05) or Insufficient Encryption (0x0f)：当发起方想要访问通过"encrypted read/write" 或 "authenticated read/write"权限访问配对设备的属性，如果连接没有加密，配对设备会发送上述错误代码。如果属性的权限是 "authenticated read/write"，并且连接仅被未认证、无MITM保护的密钥加密，配对设备也会发送0x05错误代码。当Android手机蓝牙服务收到0x05或0x0f时，会自动开启重新配对机制，忽略之前采用的配对方法。

**（4）Flaw 4 -  没有能够以编程的方式移除可疑/损坏的连接并重新配对的机制**

第三方Android应用不能从手机绑定设备列表中解除一个绑定关系，即使用户可以手动通过系统设置应用移除一个绑定。

### 4 Downgrade Attacks

#### 4.1 Threat Model

**（1）Threat model for Android mobiles**

对于移动设备，攻击有以下前提假设：

- 攻击者可以获得和受害者相同的设备来检查其应用和通信协议
- 攻击者不能物理访问手机
- 不需要在手机上安装恶意软件
- 攻击之前，Android手机和配对设备使用安全配对方法（Passkey Entry和Numerical Comparison）进行配对

如果Android和配对设备没有配对或使用Just Works配对，攻击也可以实施。

**（2）Threat model for peer devices**

对于BLE设备，攻击有以下前提假设：

- 攻击之前，Android手机和配对设备使用安配对方法（Passkey Entry和Numerical Comparison）进行配对

- 攻击者可能可以物理接触BLE设备。例如，很多IoT产品都被放置在室外。

  作者考虑两种攻击场景：

  - 攻击者能够物理接触受害者的BLE设备
  - 攻击者不能物理接触BLE设备

#### 4.2 攻击概述

对于移动设备（手机）的攻击需要：

- sniffer：嗅探器用于嗅探BLE通信，收集一些基本信息（MAC地址、名称）
- fake BLE device and fake mobile
  - fake BLE device模拟受害者设备，攻击者使用sniffer获得设备的MAC地址和名称，将伪造的设备配置为相同的MAC和名称。
  - fake mobile模拟受害者手机，攻击者需要知道受害者手机的MAC地址和IRK。
- blocker：能够实施DoS攻击，阻止受害者BLE设备连接到手机，使得伪造的设备可以连接到受害者手机。blocker可以由以下方法实现：
  - blocker可以为一个发起方，由于受害者设备的连接设备通常被限制为1，因此当blocker连接到BLE设备，其他的手机就无法连接到该设备；如果设备允许多个连接，则可以使用多个blocker。
  - 如果受害者设备使用了白名单，blocker可能会无法连接到它。针对这种情况，伪造的BLE设备可以提高其广播频率，则会有比受害者设备更高的概率连接到受害者手机。
  - 可以使用jammer（干扰器），本文未使用。

#### 4.3 Attacks against Android Mobiles

图3给出了每类攻击的步骤，以及不同攻击之间的联系。

![](/images/2020-08-10/fig3.PNG)

**（1）Attack I - False data injection via Design Flaw 3**

Fake BLE device发送Pin or Key Missing (0x06)错误代码，Android手机和fake device之间降级为明文通信。在手机访问fake device的属性时，攻击者就能够向手机中注入错误的数据。

**（2）Attack II - Spoofing attack on sensitive information via Design Flaw 3**

攻击者把通信过程降级到明文通信，fake device就可以接收来自Android手机的敏感信息。

**（3）Attack III - Stealing Android mobile's IRK and MAC address via Design Flaw 1, 2, 3**

为了防止MAC地址被泄露，使用API 23及以上的 Android手机默认使用IRK来对其进行保护。IRK在手机第一次配置的时候生成，直到手机被恢复出厂设置都不会更改，任何和手机配对的BLE设备都会收到相同的IRK和MAC地址。

-  fake device会发送0x06错误代码，使得通信被降级为明文
- 攻击者配置fake device的属性权限为encrypted read/write。当Android 应用尝试访问这些属性时，fake device会发送一个"Insufficient Authentication (0x05)"或"Insufficient Encryption (0x0f)"错误，这会使得手机开启重新配对过程。
- 然后，fake device会被配置，使得受害者手机和其使用Just Works进行配对，之后手机会传输IRK和MAC地址即可获得。

**（4）Attack IV - Denial of Service (DoS) via Design Flaws 1, 2, 3 and 4**

- 攻击者首先执行Attack III来窃取手机的MAC地址和IRK，并能够将一台fake device和受害者Android手机使用Just Works进行配对。
- 然后，攻击者关闭伪造设备和blocker，受害者手机会尝试与受害者设备进行通信。然而，手机上的LTK和受害者设备上的LTK已经不同了，但是作者发现Android不能够检测到这种不一致性，并会使通信陷入死锁。
- 由于app不能移除手机上的绑定或重新配对，死锁仅能被用户手动移除解决。

#### 4.4 Attacks against Peer Devices

目前SCO模式有两种可以用来保护一台设备敏感数据的方法：配对和属性权限。安全配对保护通信，属性权限基于采用的配对方法限制对属性的访问。作者发现属性权限经常会被误用。

**（1）Attakc V - Passive eavesdropping attack**

在受害者设备仅有read/write权限的属性时，攻击生效。

攻击者首先锁定受害者设备，然后fake device发送"Pin or Key Missing (0x06)"错误，使得受害者手机与其的通信降级为明文，之后fake device离线并且关闭blocker。这时作者发现手机会和受害者BLE设备使用明文通信，并且能够访问其read/write属性。这样，攻击者就能够窃听通信，并且使用sniffer获得到敏感信息。

**（2）Attack VI - Bypassing the whitelist**

一台BLE设备可能使用MAC地址白名单和IRK的方式，仅允许已经配对的手机进行连接。由于攻击者能够窃取到受害者手机的MAC地址和IRK，fake mobile就可以绕过白名单并和设备连接。

**（3）Attack VII - Data manipulation**

一旦fake mobile连接到设备，它可能尝试访问敏感服务。

但如果BLE设备属性的权限是encrypted read/write或authenticated read/write，fake mobile必须首先和设备进行安全配对，可能导致攻击失败。

**（4）Attack VIII - MITM attacks**

如果Attack VII能够执行，那么MITM攻击也可以实施。fake device连接到Android手机，fake mobile可以连接到设备。这样，fake device和fake mobile能够通信，并作为MITM操控受害者和移动设备之间的消息。

### 5 Evaluation

#### 5.1 Experiment Setup

实验设置：

- Adafruit Bluefruit LE Sniffer - 嗅探BLE通信

- Texa Instruments (TI) CC2640开发板 - 模拟blocker、fake BLE设备、fake mobile

- 5个手机，Android版本从7.0到9.0

  ![](/images/2020-08-10/table4.PNG)

- 18个流行的BLE产品，3个CC2640开发板

  ![](/images/2020-08-10/fig5.PNG)

数据集：

- Androzoo数据库中的18929个Android BLE apps

#### 5.2 Attacks against Mobiles

（1）对于不同Android手机的攻击普适性

作者在这些手机上测试了所有的逻辑缺陷，覆盖Android 7.0到9.0。

在Android 7.0中，fake device还可以发送一个安全请求给受害者手机，来隐藏配对过程。对于更高版本的Android设备，该请求会弹出一个配对请求对话框来要求用户进行配对，这可能会引起用户注意。

（2）对于BLE应用的攻击普适性

作者提到Android BLE编程框架存在四种设计缺陷，所有使用该框架的Android BLE 应用都会受到攻击。

为了了解应用是否使用intents来在配对结束后确定配对方法，作者基于soot开发了一个工具——BLE pairing scanner (BLEPS)来模拟app中使用的函数、构造调用图以及确定app是如何执行配对和使用intents的。

![](/images/2020-08-10/table5.PNG)

表5显示了有2005个app使用ACTION_BOND_STATE_CHANGED intent来检查手机是否和设备进行绑定，239个app同时使用ACTION_BOND_STATE_CHANGED和ACTION_PAIRING_REQUEST。

在这239个app中，有152个使用intents来确定是否使用Passkey Entry或Numeric Comparison，对于前者，这些app之后会被通过setPin()自动输入一个固定的passkey；对于后者，这些app会被通过setPairingConirmation()来点击确认按钮。有87个app注册intents用于调试，通过Log.d()打印配对状态。作者执行人工分析来检查包含两个intents的app，发现都没有实现配对状态的及时反馈。

（3）对于手机和的BLE设备上应用的攻击

作者在18个BLE产品上实施了Attacks I-VIII，结果如表6所示。

![](/images/2020-08-10/table6.PNG)

#### 5.3 Attacks beyond Mobiles

（1）对BLE设备的攻击

- 缺少SCO模式。所有181个BLE设备均未开启SCO模式，攻击者可以不需要进行物理接触，而直接使用Just Works方式与设备进行配对。
- 滥用权限。13台设备将其属性配置为read/write，攻击者可以直接访问属性而无需配对。
- SCO模式不正确实现。在TI的SDK中，尽管允许应用设置SCO模式标志位，但它仅检查配对请求是否启用了安全连接SC，而不检查协商的配对方法是否为 Passkey Entry或Numerical Comparison。
- 属性权限的不正确实现。LTK可以是由Just Works 创建的无认证、无MITM保护的密钥，或是由Passkey Entry、Numeric Comparison和OOB创建的认证、有MITM保护的密钥。假设用户手机和设备进行了配对并生成了认证的、有MITM保护的LTK，作者发现当fake mobile使用Just Works与受害者设备配对时，TI的BLE stack不会更新密钥属性，生成的LTK仍然是那个认证且有MITM保护的密钥，fake mobile能够使用authenticated read/write权限来访问属性。

作者在TI CC2640、CC2640R2F、CC2650上测试并验证了这些漏洞，并在18类BLE产品上实施了示例攻击。

（2）最大攻击距离

尽管BLE是短距离通信，对BLE设备的攻击距离取决于类似天线增益和相关设备的发射功率等因素。攻击者可以使用大天线来增加攻击距离。图6给出了针对20种不同的Android手机（包括GooglePixel4，SamsungS10和HUAWEI P30 Pro）以及图5中的18种设备的最大攻击距离的累积分布函数（CDF）。针对移动设备的攻击距离平均值和最大值分别为77.2米（m）和94.0m ，以及针对BLE设备的为46.5m和77.1m。

![](/images/2020-08-10/fig678.PNG)

（3）键盘连接竞争

当受害者键盘和伪造键盘尝试连接到受害者手机时，具有较高广播频率的键盘具有更大的机会。作者测试了广播频率对伪造键盘成功连接到受害者手机的影响。在正常使用情况下，受害者键盘被放置在靠近Android手机的位置，而伪造键盘则距离键盘10米图7显示了成功率与广播频率的关系。伪造键盘广播频率为30HZ时，成功率为50％。 BLE规范将最高广播频率设置为50HZ，在该频率下，伪造键盘的成功率为75％。

### 6 Flaw and Attacks in other OSes

![](/images/2020-08-10/table7.PNG)

由于论文是针对蓝牙规范的缺陷，作者发现这些提到的问题在其他主要OS（包括iOS，macOS，Windows和Linux）中也存在。表7比较了针对不同最新版本的操作系统和设备的设计缺陷以及攻击。

作者总结了以下几点不同：

- 一个特定的OS可能不会有四种缺陷。
- 一些OS在配对之后可能知道所采用的配对方法，但是一些操作系统可能不知道。
- 一个OS可能没有Flaw 3，但是允许应用来处理错误；但是所有OS都具有Flaw 1和2，app处理错误的方式也不同。
- 一些计算机操作系统默认没有IRK，Linux设备采用IRK。没有IRK的保护，攻击者可以嗅探BLE连接，获取设备MAC地址，部署攻击。