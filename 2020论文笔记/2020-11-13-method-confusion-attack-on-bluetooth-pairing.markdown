---
layout: post
title: "Method Confusion Attack on Bluetooth Pairing"
date: 2020-11-13 15:39:50 +0800
comments: true
categories: 
---

> 作者：Maximilian von Tschirschnitz, Ludwig Peuckert, Fabian Franzen, and Jens Grossklags
>
> 单位：Technical University of Munich
>
> 会议：Oakland 2021
>
> 链接：https://www.computer.org/csdl/pds/api/csdl/proceedings/download-article/1mbmHzm2Q6c/pdf

本文作者发现了蓝牙配对机制中的一个设计缺陷，能够让攻击者实现MITM来使用不同的方法来与原来要进行配对的两台设备配对，而在交互时受害设备不会意识到这种方法混乱（Method Confusion）。与其他攻击相比，该攻击即使在蓝牙最高安全模式中也可以实现，并且无法在协议实现中解决。通过该攻击，攻击者能够渗透到受害者的安全连接中，并拦截所有流量。

作者将该攻击成功的应用于多个场景中，实现了端到端的PoC，并使用了手机、智能手表、银行设备来进行测试，均能够在用户未察觉的情况下攻击成功。

<!-- more -->

## 1 INTRODUCTION

BLE一些配对方法没有验证是否双方设备实际上使用相同的方法来完成配对，因此，可能存在两个独立的配对过程，其中双方设备使用不同配对方法与对方交互。即使用户参与到配对过程，也无法提供足够的信息来识别这种方法混乱。

本文作者展示了Method Confusion Attack：攻击者首先拦截并劫持两台设备的配对消息，然后分别对两台设备实施不同的配对方法，设备以为是与期望的设备正在进行配对。这样攻击者就可以获得机密信息，来影响配对过程以成功配对，实现MITM攻击。尽管受害者认为已经建立了可信任的连接，但他们却不知道正与攻击者配对，攻击者现在处于稳定的中间人（MitM）位置。 

Method Confusion Attack本身并没有破坏每种配对方法的加密过程，因此现有的蓝牙安全机制无法缓解该问题，只能修改BLE规范缓解它。

主要贡献：

- 作者提出了Method Confusion Attack，攻击者能够通过一个设计缺陷来进行MITM攻击
- 作者展示了攻击能够影响百万台设备（智能手机、智能手表以及银行设备）
- 作者讨论了在某种特定的情况下攻击可以被缓解，作者进行了用户调查，发现没有参与者注意到该攻击。
- 作者为设备供应商和蓝牙规范提出了修复策略

## 2 Background

由于LE Legacy Pairing 存在很严重的安全问题（[BIAS](https://francozappa.github.io/about-bias/publication/antonioli-20-bias/antonioli-20-bias.pdf)），因此本文作者仅针对BLE 4.2之后的LE Secure Connections。

### 2.1 LESC Pairing Process

![](/images/2020-11-13/uploads/upload_958c21d4c53330fac59814b73bfde24a.PNG)

主要流程：

（1）Pairing Feature Exchange：配对设备互相交换安全要求和I/O能力。

（2）Public Key Exchange：交换ECDH公钥信息并计算DHKey。由于双方没有认证公钥，因此生成的DHK也是不可信的。

![](/images/2020-11-13/uploads/upload_fe563993421f7f658c55880ca05ad7e2.PNG)

（3）Authentication：认证方法与配对模式（Association Model）有关，配对模式由双方交换的特征决定。两台设备独立的进行配对模式选择，该过程之后，两台设备会认为他们选择了相同的方法并且对PK的身份进行了验证。

（4）LTK Calculation and Validation：使用认证的PK来建立一个安全的信道，并生成LTK。
$$
MacKey||LTK = f_5(DHK,N_I,N_R, addr_I, addr_R)
$$
（5）Confirmation：双方验证生成的LTK

![](/images/2020-11-13/uploads/upload_05c23296979ef9afeef962f8e5dce5c6.PNG)


### 2.2 Association Model Agreement

BLE绑定过程包含四种方式：

- Out of Band：PK通过其他信道方式认证，例如NFC、QR-Code
- Just Works (JW)：无任何认证机制，直接配对
- Numeric Comparison (NC)：双方需要比较屏幕上的6位数字是一致的
- Passkey Entry (PE)：用户需要在一台设备上输入另一台设备显示的6位数字

配对过程中Pairing Feature Exchange用于协商配对方法，与三个特性有关：

- OOB-bit：表示使用OOB数据配对
- MitM-bit：表示认证要求
- IOCaps：表示用户界面提供的IO能力

接下来设备根据这些特性决定使用什么样的配对方法：

- 如果每台设备都有OOB数据，那么OOB-bit被设置为1，两者使用OOB认证方法进行配对。本文攻击不适用于OOB模式，因此不做讨论。
- 如果没有设备设置MitM-bit，那么双方使用JW进行配对。由于JW没有提供MITM保护，因此本文也不做讨论。
- 如果MitM-bit被设置，则通过IOCaps来决定使用NC还是PE配对，如图5所示。

![](/images/2020-11-13/uploads/upload_2e65e6298451899de9586554aaa50f6f.PNG)


## 3 METHOD CONFUSION ATTACK

Method Confusion Attack在配对尝试阶段（pairing attempt，即配对开始之前的广播阶段）进行攻击来实现MITM，攻击者会与双方设备R (-> MI)和I (-> MR)分别同时进行两个配对过程，一个通过NC方法配对，另一个通过PE方法配对。

攻击的合理性如下：

- NC和PE都是用了相同格式的数字，所有他们不能分辨出一个给定的数字是由NC配对方法产生还是PE。
- 设备没有验证他们的配对设备实际上使用了什么配对模式。
- BLE规范没有提供能让用户知道所使用配对模式的方法。

### 3.1 Attack Preparation

（1）Initiator Connection to MITM

首先I需要和MR（MITM Responder）开启配对，作者假设用户尝试将I和R两台设备进行配对，R开始广播自身信息，I搜索广播消息。

同时MR也进行广播（名称与R相同），用户看到MR在I的配对名单中，并且MR无法与R进行区分。为了避免R出现在列表上，R的广播信号会被屏蔽。

（2）MITM Interaction during Attack

在配对请求到达MR时，MI开启和R的连接。现在由于I和R都仅和MITM通信，所有配对过程的通信都由MR和MI处理。

基于攻击设备的IO能力，攻击有两个变种：Passkey on Numeric (PoN) 和Numeric on Passkey (NoP)。

### 3.2 Passkey on Numeric

在PoN，I与MR执行PE配对，R和MI执行NC配对。图6显示了I和R具体的交互过程。

![](/images/2020-11-13/uploads/upload_22255ddfb36c4d4878704f54b200234a.PNG)


（1）Pairing Feature Exchange

- I开启与MR的配对，并且传输其安全要求和IO能力（Keyboard*）
- MR响应I，并且传输安全要求（设置MITM-bit）和IO能力（DisplayOnly）

这时，MI开启与R进行特征信息交换：

- MI开启与R的配对，并传输安全要求（设置MITM-bit）和IO能力（DisplayYesNo）
- R响应MI，并传输安全要求和IO能力（DisplayYesNo/DisplayKeyboard）

（2）Public Key Exchange

（3）Authentication：

- MR暂停和I的配对过程，同时MI开始执行和R的NC-based authentication过程
  - 交换参数后，MI和R计算Va,Va为之后显示在R上的6位数字
  - 等待用户比较数字完成NC认证
- 之后开启MR和I的PE认证过程：
  - MR设置口令rb = Va，并将其发送到I
  - I要求用户输入6为数字来执行PE认证

总的来说，用户视角来看相当于一个合法的PE配对模式，I要求用户输入6位数字，并且R上显示了6位数字等待确认。

（4）LTK Calculation and Validation

之后I和MR计算DHK(I,MR)完成认证并交换确认信息，完成配对过程，并建立相同的LTK。同样的，MI和R也进行类似的操作并完成配对。

这样，M就能够重放所有I和R之间的信息，解密收到的消息并将其重新加密并转发，但是I和R却不能察觉。

### 3.3 Numeric on Passkey

NoP与PoN类似，最大的区别是I和MR之间使用NC配对，R和MI之间使用PE配对。

由于过程类似，这里也不做过多赘述。




## 4 IMPLEMENTATION

为了验证此类攻击在现实生活中的可行性，作者设计了一个端到端的攻击框架，该框架包含三个组件：

- Method Confusion Attack implementation
- Jammer
- Address sniffer（LE Privacy开启，使用随机地址）

### 4.1 BThack and MITM Application

作者在蓝牙MITM平台**BThack**上实现了Method Confusion Attack。该平台基于BTstack（一个BLE协议栈），允许应用更改BLE通信过程。作者将其与**USB Cambridge Sililcon Radio, Bluetooth Dongles (CSR dongles)** 结合，通过回调机制来中断控制流。

BThack MITM应用包含两个内存独立的进程，分别包含独立的BLE栈，其中一个进程扮演R的角色，另一个扮演I的角色，两者通过IPC通信。

PoN攻击如之前所述，I连接到MR，反过来触发MI连接到R。在MR的passkey生成过程存在一个回调，等待MI和R完成NC后，设置Va为passkey。NoP类似。

### 4.2 Selective Jamming

为了能让MR与I连接，而不是R，我们需要阻止R的广播报文送达I，因此作者实现了一个可选择的广播报文屏蔽。

首先，屏蔽器需要能够识别广播报文包并且能够**有选择性的干扰**，并且干扰需要低延时。

本文作者采用了前人的工作，使用nrf51 BLE芯片上的定制固件。该芯片有一个**地址匹配功能**，能够配置一个4字节的模式，如果发现，会触发数据包接收过程，如果匹配到特定数据包的报头地址，则会停止数据包接收，并切换到传输模式，传输虚拟包，改变有效载荷并导致广播分组的校验和不匹配，从而导致设备在接收时丢弃该分组。这些都是在无线电硬件中执行，无需CPU参与以降低延迟。

此外，如果LE Privacy不开启，则广播地址是固定的，我们匹配广播地址的前四字节即可；如果LE Privacy开启，广播地址随机，作者认为可以通过匹配Local Name来实现相同的效果。

## 5 Evaluation

作者实现了端到端的PoC，并设计了几个实验来测试攻击。

- 首先作者进行了测试来检验实际Method Confusion Attack的干扰功能
- 然后作者进行了完整的端到端实验来证明Method Confusion Attack能够完成MITM
- 之后作者在真实设备上进行了测试
- 最后作者进行了性能评估实验来衡量吞吐量和延迟。

实验环境：

- 作者实现了三个运行在CSR dongles上的BTstack应用来模拟设备R，I，M
- 作者使用了micro:bit设备来实现干扰功能，每个开发板都装配一个nrf51无线电模块

### 5.1 Testing Jamming of Bluetooth Advertisements

作者通过一系列重复实验来验证干扰功能，包括开启LE Privacy和不开启LE Privacy。

作者将R配置为可发现的模式，并在开启干扰系统的时候尝试与R配对。

实验结果表明在所有的测试中，没有来自R的广播消息到达I。

 ### 5.2 End-to-End Lab Test

由于Method Confusion Attack有两种变形，作者测试了两个场景，唯一区别是R的IO能力不同。

- Scenario 1: R的IO能力是DisplayYesNo
- Scenario 2: R的IO能力是KeyboardOnly

在两个场景中攻击者使用之前的端到端攻击框架来进行Method Confusion Attack以实现MITM。

结果表明，两种场景都可以成功的实现MITM攻击，之后攻击者就能够窃听机密信息以完成配对。

### 5.3 Production-Device Evaluation

作者使用了一些真实设备来测试NoP和PoN：

- I设备为OnePlus 7 Pro (Android 9.0)、iPhone 11 with iOS 13.4.1以及Thinkpad W540 (Windows 10)，都具有DisplayKeyboard功能
- 两台R设备为
  - 1）Samsung Galaxy Watch 42mm，Tizen Wearable OS 4.0，DisplayKeyboard，NC配对，攻击策略为PoN；
  - 2）Reiner SCT tanJack Bluetooth，wireless TAN-Generator，KeyboardOnly，PE配对，攻击策略为NoP。

其他实验过程与Lab Test相同，攻击者能够完成和受害者设备的配对并且实现MITM。

### 5.4 Performance Evaluation

作者测试了吞吐量和RTT（往返时间）以评估网络性能对于攻击效果的影响。

![](/images/2020-11-13/uploads/upload_d51e3d502153ee2c0a6eb8acbca85726.PNG)


结果如图7所示，MITM与正常连接的平均性能几乎相同，因此对于大多数应用，所达到的连接质量已经足够了（不超过300ms RTT）。

## 6 ROLE OF THE USER IN THE ATTACK

作者进行了一系列调查来说明攻击是实际可行的，即用户无法察觉。

### 6.1 User Model

在Method Confusion Attack，一台受害设备显示6位数字并想要用户比较确认数字，同时另一台设备想要用户输入6位数字。

BLE规范对于这些交互并没有提供任何规则，实际上有很多种设计和实现，如图8所示。

![](/images/2020-11-13/uploads/upload_468613366e74cd60349160c9d351bd09.PNG)


### 6.2 Chances of Detection

用户可能可以发现两台设备执行了两个不同的配对模式，但这不太现实：

- 普通用户通常不清楚蓝牙规范会向他们请求哪些操作
- 因为没有明显的提示，用户很难发现使用了不同的关联方法，尤其是NC和PE
- 用户很难区分合法配对和被攻击过的配对之间的区别

作者对35种流行设备进行了调查，这些设备都没有将PE显示和NC明显区分出来，用户也不太可能识别出问题。

为了进一步验证，作者进行了一项用户调查，选择了40位受过高等教育并有技术研究经验的参与者。实验中包含三种常见的支持蓝牙的设备和支持蓝牙的手机：索尼WH-1000XM3（OOB/NFC配对）、SoundPeats TrueCapsule入耳式耳机（JW配对）、Reiner SCT tanJack Bluetooth（PE配对）、Samsung Galaxy S8 Edge - Android 9.0 （支持所有LESC配对方法）。随后，测试用户使用这些无线设备。

![](/images/2020-11-13/uploads/upload_1139b92b346b59eb2b5d06191fe7dfaf.PNG)


实验结果表明，有37位（92.5%）的参与者完成了配对过程，并且攻击者实现了MITM；对于其余3位参与者没有完成配对。40名参与者均未察觉出攻击。



## 7 Summary

### 7.1 Proposed Fix

作者提出了一些修复策略：

（1）强制使用某种配对方法。在某些情况下，厂商可以假设配对设备的某些确定的属性，从而可以通过设置IOCaps来是配对方法固定。

（2）用户提醒。虽然蓝牙标准没有指定提醒用户的方法，但是厂商可以通过在配对时提醒用户，例如弹出不要输入数字等对话框。

（3）认证关联模型。将所使用的配对方式融入到给用户显示的信息中。比如说PE使用的密钥和NC必须有明显的区别，PE使用字母而NC使用数字。

### 7.2 总结

- 作者提出了一种新的针对BLE标准的攻击——Method Confusion Attack，即使在最安全的模式下，攻击也能够实现，并且用户很难察觉。

- 总结一下目前针对BLE的研究，大多专注于BLE标准的问题，包括传统连接与安全连接、初次连接与重连；还有一些针对从BLE实现中发现问题。

  - [BIAS](https://francozappa.github.io/about-bias/publication/antonioli-20-bias/antonioli-20-bias.pdf)：传统连接无双向认证，安全连接降级问题，主从角色切换问题
  - [Method Confusion](https://www.computer.org/csdl/pds/api/csdl/proceedings/download-article/1mbmHzm2Q6c/pdf)：安全连接MITM
  - [Secure Pairing Downgrade Attacks](http://jin.ece.ufl.edu/papers/USENIX2020-BLE.PDF)：安全连接SCO中存在的实现缺陷，能够导致降级攻击
  - [BLESA](https://www.usenix.org/conference/woot20/presentation/wu)：BLE重连机制缺陷，会导致Spoofing Attacks
  - [KNOB](https://github.com/francozappa/knob)：针对BLE加密密钥协商过程，降低会话密钥的熵值
  - [Bluetooth Randomness](https://www.usenix.org/system/files/woot20-paper-tillmanns.pdf)：蓝牙芯片中的随机数生成问题
  - [FirmXRay](http://web.cse.ohio-state.edu/~lin.3021/file/CCS20.pdf)：检查固件来发现BLE问题
  - [BLEScope](https://doi.org/10.1145/3319535.3354240)：从app中检测静态UUID问题
  - [BlueShield](https://www.cs.purdue.edu/homes/dxu/pubs/RAID20-BlueShield.pdf)：检测Spoofing Attacks中的虚假广播包


  

















