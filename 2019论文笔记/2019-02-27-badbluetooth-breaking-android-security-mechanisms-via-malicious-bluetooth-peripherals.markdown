---
layout: post
title: "BadBluetooth: Breaking Android Security Mechanisms via Malicious Bluetooth Peripherals"
date: 2019-02-27 16:37:43 +0800
comments: true
categories: 
---

出处: NDSS 2019

作者: Fenghao Xu; Wenrui Diao; Zhou Li; Jiongyi Chen and Kehuan Zhang 

单位: The Chinese University of Hong Kong

原文: https://staff.ie.cuhk.edu.hk/~khzhang/my-papers/2019-ndss-bluetooth.pdf

演示: https://sites.google.com/view/bluetoothvul

<hr/>

这篇论文针对Android蓝牙协议栈实现的粗粒度权限管理策略提出了一种新型的攻击方式BadBluetooth，攻击者通过将蓝牙设备伪装为键盘，网络接入点和耳机，同时配合Android发起静默配对，最终实现控制手机截屏偷取用户隐私数据，劫持通信流量，甚至在锁屏状态下拨打电话等。

<!--more-->
## 背景
### A. Bluetooth概述
蓝牙作为WPAN技术的代表已经有20多年的历史，广泛应用于PC、手机、智能设备等，蓝牙标准由Bluetooth SIG管理, 现在已经发展到蓝牙5.0。

#### Bluetooth Stack
蓝牙协议栈设计通信多层架构，包括物理层、链接层、中间层和应用层。

- L2CAP管理两个设备的连接

- RFCOMM用于串行数据传输

- SDP是广播设备支持的服务

- GATT用于低功耗蓝牙模式

- ...

![](/images/2019-02-27/media/15504752248738/15504763159464.jpg)

#### Bluetooth Profile
- 代表着设备的功能
- 30多个种类 [List of Bluetooth profiles](https://en.wikipedia.org/wiki/List_of_Bluetooth_profiles)
    - Headset Profile	
    - Human Interface Device Profile (HID)
- 每个设备可实现多个profile 

#### Bluetooth Connection
- 发现阶段，扫描发现附近设备，包括设备名字，设备种类，设备profile
- 配对阶段，致辞多种配对模式，一般需要用户输入pin码或者比较数据
- 建立连接，两个设备配对后共享link key，用于加密双方通信的数据

### B. Android Bluetooth
从Android 4.2， Google开发了自己的协议栈Bluedroid （Fluoride）

#### Bluetooth Permission
- normal-level，无需用户确认，用来请求和接收连接
  - BLUETOOTH
  - BLUETOOTH_ADMIN
- dangerous-level， 需要用户授权，扫描附近设备，用来获取用户位置
  - ACCESS_COARSE_LOCATION
  - ACCESS_FINE_LOCATION
- signature-level，需要用户授权，用户需要交互的配对过程
    - BLUETOOTH_PRIVILEGED
## 设计缺陷
由于蓝牙协议栈的安全设计是粗粒度的，以设备为级别的，作者列出了5个Android实现的蓝牙协议栈的缺陷。

-   Weakness #1: 配置文件上的身份验证过程不一致
当两个设备进行配对时，并未验证配置文件。以Android为例，之后配对后用户才可见。如果设备更改了Profile，它仍然受到信任，并且不会通知用户。 假如用户开始连接了一个伪造的耳机设备，攻击者把它的profile改为HID后，就可以向手机静默的发送按键消息。

![](/images/2019-02-27/media/15504752248738/15508231919197.jpg)

- Weakness #2: 对配置文件连接的过度开放
蓝牙协议栈支持许多配置文件，一旦双方配对完成，Host会尽力连接到声明这些配置文件的设备，即使用户可以稍后在设备详细菜单中断开某些配置文件，这样的决定不会被host记住。

- Weakness #3: 可欺骗的用户界面
当用户浏览蓝牙设备列表时可以看到设备的名称和图标。作者发现只要改变CoD号码，攻击者就可以选择要呈现的图标。另一个问题是缺少蓝牙相关信息。例如，只有两个事件会在通知栏中提示蓝牙：显示蓝牙已打开，显示已连接远程设备。

- Weakness #4: 与设备静默配对
当从设备端发送配对请求时，Android系统会弹出对话框让用户确认。但是，如果由手机发起此过程，则可能没有通知。 比如，当设备没有显示能力或输入能力（例如，耳机），配对会使用“Just Works”模式，因此可以利用此功能进行静默配对。

- Weakness #5: 没有对配置文件的权限管理
Android通过权限限制应用程序是否可以访问蓝牙设备。但是，这样的权限管理太粗糙，与配置文件不一致。例如，关于蓝牙键盘的配置文件（即HID）应该只能由系统进程访问。但是，当第三方应用程序被授予BLUETOOTH_ADMIN权限时，键盘可自动访问。尽管官方已经注意到了该问题，并且通过删除相关的public API代码。但是，作者通过Java的反射机制，利用第三方app实现了访问这些profile的功能。

## 攻击概述
### 敌手模型
假设:
####  手机上安装具有Bluetooth权限的恶意app
    - BLUETOOTH和BLUETOOTH_ADMIN是一般权限
    - 无需请求用户同意权限申请
#### Bluetooth设备是受攻击者控制的
    - XcodeGhost攻击
    - 通过设备其他漏洞获得设备权限后插入恶意代码

### 攻击流程

![](/images/2019-02-27/media/15504752248738/15508150853196.jpg)

#### 攻击原语
- Changeable Profile (Weakness #1, #2, #3)
当配对完成后，设备添加其他配置文件，并在攻击完成后将其删除。 用户很难发现这一变化，除非重置配对或者添加新的配置，设备详细菜单不会发生变化。

![](/images/2019-02-27/media/15504752248738/15508160017982.jpg)

- Silent Pairing (Weakness #4)
    - setPairingConfirmation 需要申请 --BLUETOOTH_PRIVILEGED-- 权限 （不使用）
    - 恶意蓝牙设备使用“Just Works”配对模式 
- Connecting Sensitive Profile (Weakness #5)
攻击需要利用手机上的敏感profile，这些是第三方app实现的。这些app实现了proxy class来操作敏感profile，并把IPC binder封装到蓝牙系统中。攻击者使用framework.jar而不是android.jar就可以调用私有类和方法，比如如下代码中调用getProfileProxy

![](/images/2019-02-27/media/15504752248738/15508164812449.jpg)

#### 攻击阶段
为了欺骗用户设备是安全的，设备假装自己是温度传感器，配对将静默运行。设备使用SPP配置，并且设备细节菜单不会显示任何profile。

1. 启动恶意app，并保持后台运行，直到监测到手机屏幕关闭时开始攻击
2. 通过调用BluetoothAdapter.enable静默配对已知地址的恶意设备
3. 设备等待从app发来的命令，命令通过蓝牙信道传送，或通过网络转发
4. 接到命令后，设备使能敏感的配置文件，App利用存在的配置文件功能
5. 设备恢复正常的状态，App使用removeBond取消配对，以免引起注意

## 攻击

### Exploited Profiles
Android目前支持的Bluetooth Profile和对应的使用场景。作者发现HID，PAN和HFP/HSP这三种Profile可以被攻击者滥用。所有的攻击演示可以在[这里](https://sites.google.com/view/bluetoothvul/)访问。

![](/images/2019-02-27/media/15504752248738/15508420626884.jpg)

### A. Human Interface Device

HID用来连接输入设备，比如鼠标和键盘。手机可以通过这些外设被控制，同时Android提供了功能齐全的键盘和鼠标的支持，比如点击鼠标等同于用手指触屏操作。当HID被接入后，所有运行的App和主屏幕都能接受到输入事件。

![](/images/2019-02-27/media/15504752248738/15508423978038.jpg)

#### HID Report
数据格式

![](/images/2019-02-27/media/15504752248738/15508427873697.jpg)

#### 攻击策略
-   自适应攻击，如何识别鼠标的位置，一方面可以通过恶意App收集系统版本，进而实施不同的攻击载荷。另一方面，攻击设备可以接收和手机相关的信息。初始的时候将鼠标移到左上角。
-   输入能力
通过HID input report，即可以模拟按键和鼠标点击。Android定义了更多的功能键，比如Home、Back和Volume Control。恶意设备发送构造好的输入事件达到输入攻击载荷的目的。

![](/images/2019-02-27/media/15504752248738/15508432024157.jpg)

-   输出能力
通过截屏，或者通过选择文字并复制粘贴，达到输出的能力。

#### 攻击危害
-   信息窃取
通过截屏，偷取邮件，短信，通讯录等等。比如申请WRITE_EXTERNAL_STORAGE权限，然后通过网络发送图片。或通过浏览器上传等等。

-   操控App和系统
基于Android的布局和图标位置信息判断手机品牌和版本，并在本地或远程执行图像分析。作者通过实验，持续发送KEY POWER模拟长按电源按钮，此时系统会弹出电源管理菜单。之后，通过先前的知识可以选择关机或重启。

-   其他
如果攻击者控制了手机，他可能会盗取短信中的验证码或是登录网站记住的密码。 他也可能盗用受害者身份，发送垃圾邮件，甚至可以打开相机捕捉周围环境。

### B. Personal Area Networking

![-w438](/images/2019-02-27/media/15504752248738/15509931085009.jpg)


攻击危害
-   网络嗅探和欺骗
手机可以通过蓝牙设备访问互联网，所以可以设备端提供NAP服务并执行中间人攻击。 在这次攻击中，设备启用NAT服务，一旦手机连接，蓝牙设备会收到所有来自手机的以太网数据包，然后就可以在设备端拦截的所有流量。

-   网络消费
手机可以充当NAT并通过蓝牙共享其网络资源。在此攻击中，设备声称其身份为PANU，并尝试共享手机网络。 开放蓝牙网络共享不需要任何特权。使用的API是
BluetoothBan类的setBluetoothTethering函数。

### C. Hands-Free

![-w433](/images/2019-02-27/media/15504752248738/15509931189376.jpg)

-   电话控制
HFP定义了两个角色 - Audio Gateway（AG）和Hands-Free Unit（HF）。 此次攻击中，设备声明为HF，并等待电话连接。设备可以发送命令回答，拒绝或终止来电。 更重要的是，蓝牙设备能够拨打任意号码。

-   语音命令注入
HFP还可以触发Google Voice Assistant。 默认情况下，Google允许手机在锁定的状态下仍然可以通过蓝牙耳机发送语音命令。 在攻击中，首先触发助手并打开音频连接， 然后就可以注入任意语音命令。 

### D. Other Profiles
除了上面三种Profile，SIM Access（SAP），短信访问（MAP），电话簿访问（PBAP）和Object Push Profile（OPP）也是潜在的目标。 但是，那些Profile要求蓝牙设备作为发起者，当蓝牙设备请求时，将会通知用户，并且手动必须批准请求，使攻击不那么隐秘。

## 实现和评估
![](/images/2019-02-27/media/15504752248738/15508144980730.jpg)

### 实验设备
-   树莓派2代（Linux OS）
-   CSR8510 USB蓝牙适配器
-   Google Pixel 2 （Android 8.1）

### 实现
Raspberry Pi 2 + 1100 行Python 代码（PyBluez）

- HID attack，raw L2CAP
- PAN attack，tcpdump和dnsmasq
- HFP attack，pulseaudio和ofono

![](/images/2019-02-27/media/15504752248738/15508147516688.jpg)

## 防御
防御框架提供对Profile细粒度的控制，阻止未授权的Profile变动。在配对时，防御框架将会弹出一个对话框，显示蓝牙设备声明的配置文件（从其SDP记录中提取）。 用户手动选择允许的Profile，该记录将被插到数据库中。 因此，我们的scheme支持用户显示地审查设备以防止静默配对行为。 每当ProfileService收到连接就验证每个设备的记录。

- Pairing Monitor
- Binding Policy DB
- Connection Controller

![-w345](/images/2019-02-27/media/15504752248738/15509973398476.jpg)

### 实现
- Pairing Monitor 

![-w237](/images/2019-02-27/media/15504752248738/15509952872569.jpg)

- Connection Controller
白名单策略，只有被ProfileService验证过的策略会记录到报名单中。

- Settings App
提高用户的可用性

### 评估
有效性和性能

![-w243](/images/2019-02-27/media/15504752248738/15509953510914.jpg)
