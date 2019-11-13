---
layout: post
title: "Dazed Droids: A Longitudinal Study of Android Inter-App Vulnerabilities"
date: 2019-01-16 16:30:50 +0800
comments: true
categories: 
---


作者：Ryan Johnson, Mohamed Elsabagh, Angelos Stavrou, Jeff Offutt

单位：Kryptowire, George Mason University

出处：ASIA CCS ’18

资料：[论文](https://dl.acm.org/authorize?N659832)

# 1 ABSTRACT & INTRODUCTION

随着Android应用复杂性的提高和功能的丰富，Android更依赖于应用之间的代码和数据共享，以缩短响应时间并提供更丰富的用户体验。 

绝大部分的Android App之间和其本身发生数据通信的时候使用的都是`intent`对象：`intent`类似于消息的抽象，提供了一种便于数据交换的基本通信机制。但是有些时候开发者们有意或无意地暴露了一些App Components内部的接口，让它们可以被本地的其他的一些App可以访问。

这些暴露的接口有些没有实现较强的error catch或数据访问控制就会导致一些安全问题。在本文中，作者发现了如下问题：DoS、提权、系统crash和数据泄露。作者设计并实现了一个Fuzzing工具去自动化地检测inter-app通信时所存在的漏洞。Daze通过null和not-null但不为空的payloads去fuzz App中`export="True"`的组件所公开的接口，并通过监控外置存储卡上文件的修改和系统设置的变化来验证漏洞是否存在。

作者分析的样本涵盖了32个不同的Android设备，AOSP4.4-8.0所有版本和18,583个Google Play上的免费App。平均每个设备分析的时间为3小时，每个应用的时间为2分钟。大约有51.7%的Android设备和Google Play上49%的Top300应用至少存在一个inter-app漏洞。
<!--more-->

# 2 BACKGROUND

Android应用被划分为组件以便于code reuse。 app组件是应用中的入口点，在app的上下文中提供特定用途。 App组件通常在`AndroidManifest.xml`中声明和静态注册。组件即四大组件：activity, service, broadcast receiver, and content provider

###2.1 Accessing App Components

其中除了content provider之外的所有组件都通过发送`intent`对象来访问。 `intent`是src应用组件发送到一个或多个dst应用组件的消息的抽象。`intent`是应用自身和其他应用通信的主要方式。`intent`的dst应用通过包名称和组件类名称来指定。每个`intent`的具体用途是通过类似如下字符串来确定的：android.provider.Telephony.SMS_RECEIVED。组件可以在`AndroidManifest.xml`中将`export`设置为`true`，这样就可以将其功能暴露给系统和设备上的其他应用程序。默认情况下，除了动态注册的`broadcast receiver`，app组件通常不会导出（只能从同一个app或相同签名的app内部中访问。）如果app的组件声明了至少一个它可以处理的操作并且没有明确声明它不应该被导出，那么Android系统将export应用程序组件。

### 2.2 Errors of Omission

小标题的`Errors of Omission`是指由于程序员的疏忽以及程序鲁棒性不足导致的这些公开的接口存在的错误。大致有以下几类（原文作者没有严格分类，因为fuzz的时候不需要指定类型，只是阐述了一些例子）：

如果目标app组件被声明了但是没有实现，那么接收app会抛出**ClassNotFoundException**错误然后应用会终止。

在处理接收的intent之前，如果所需要的native lib缺失，则会抛出**UnsatisfiedLinkError**然后终止。

……

在Android OS中组件未处理的异常可能导致关键系统进程崩溃，触发操作系统重新启动以尝试恢复。下图显示的接收intent没有carry bundle对象，则会引起Android系统的crash，这是一个存在Samsung Galaxy S6 Edge手机 Android 6.0.1 版本中的真实代码片段。![1539070489678](/images/2019-01-16/assets/1539070489678.png)

# 3 OVERVIEW OF DAZE

![https://loccs.sjtu.edu.cn/gossip/images/2019-01-16/1539071084637](/images/2019-01-16/assets/1539071084637.png)

Daze使用了8.1k行的Java和其他附属脚本实现其功能。Daze会测试所有App的四种组件并输出在测试期间的所有的相关数据。Daze使用null-fuzzing去fuzz App组件，作者选择null-fuzzing而不是data-fuzzing是因为作者认为随机的data-fuzzing只会带来1%的提升，但是每个App会使用较长的时间花费(hours per app)。

### 3.1 Identifying Statically Registered Components

Daze通过pm查询所有已安装的包，从app中提取静态注册的组件，然后遍历每个包信息。 Daze忽略未导出或需要访问权限的组件接口。  况且用户更愿意下载没有权限的应用程序。 由于Daze是开源的，因此可以轻松修改它以请求所有可用的第三方权限和测试受Android 声明的权限保护的组件。

###3.2 Identifying Dynamically-Registered Broadcast Receivers

只有**Broadcast Receivers**是所有组件中既可以动态注册又可以静态注册的。在运行时，动态注册的Broadcast Receivers只能使用操作字符串定位。Daze使用`dumpsys activity broadcasts`枚举所有动态注册的接收器。而对于第三方的App，作者使用 `adb pm grant<package> <permission>`来处理。

### 3.3 Testing Intent-Accessible Components

在定位所有静态和动态注册的组件之后，Daze通过如下步骤使用Intent进行测试。

> 1. Daze发送包含最少数据的intent到目标组件。目标组件的名称包含包名和组件的类名以及动态注册的广播接收器的操作字符串。
> 2. 如果没有发生crash，那么Daze会添加空的Bundle对象到intent然后继续发送
> 3. 如果仍然没有错误抛出，Daze会添加空的操作字符串到intent然后继续发送
> 4. 最后，发送schemeless URI。

因为bundle和uri是intent中最常用的数据内容。 为了覆盖intent可到达的所有code site，Daze会发送带有`FLAG_ACTIVITY_SINGLE_TOP`标志的intent，这样会强制把intent传递给Activity组件的onNewIntent方法。

###3.4 Testing Content Providers

和其他组件不同，`content provider`不能直接通过intent直接访问。 任何`content provider`都必须从Content Provider类中实现一组方法实现`read`和`write`数据到`content provider`。因为它们的后端是由SQLite数据库支持，在默认情况下不会导出`content provider`接口，因此往往受权限保护。Daze通过null-fuzzing Content Provider类中的所有回调方法来测试`content provider`。

测试`content provider`会比较麻烦，因为`content provider`中的崩溃会导致连接到所崩溃的`content provider`的所有app被终止。 发生3次后，Daze将停止测试`content provider`。

### 3.5 Monitoring System State

Daze通过两部分来监控系统的状态：

> 1. 通过每次fuzz前后对sdcard上所有的文件做snapshots，然后比较fuzz前后文件的变动和修改来判断敏感信息泄露
> 2. 通过查询系统属性和全局系统设置来判断是否有提权
> 3. 通过Android OS弹出的dialog box来判断当前app的crash状态，并等待消除后再继续fuzz

# 4 STUDY 1: DEVICE EVALUATION

此部分，作者测试了覆盖Android 4.4-8.0的21个厂商设备及其所有的预装app

![](/images/2019-01-16/assets/1539081244509.png)

![](/images/2019-01-16/assets/1539081222814.png)

其中有72% 的设备出现至少一次crash，32款手机的预装app中，Daze共触发了4972次crash，图D.2显示了这些设备中AOSP和厂商定制分别导致crash的次数占比

![](/images/2019-01-16/assets/1539081455566.png)

表2显示了所有设备中，crash次数前12的的app的包名，这些绝大部分都是系统关键程序，例如电话、蓝牙、UI和短信。他们可能会造成DoS。

![](/images/2019-01-16/assets/1539081525036.png)

表D.1显示了Daze测试过程中造成的系统crash：

![](/images/2019-01-16/assets/1539081654768.png)

表3显示了这些app所存在的提权漏洞：

![](/images/2019-01-16/assets/1539081714339.png)

图2为这些设备的测试时长：

![](/images/2019-01-16/assets/1539081822076.png)

# 5 STUDY 2: GOOGLE PLAY APP TESTING

此部分的测试样本为Google Play商店March 2016至April 2017的4,972个独立应用的18,583个apk文件。测试结果显示有34.7%的程序会crash，具体crash分类如下：

![](/images/2019-01-16/assets/1539082044205.png)

图6显示了这些应用中所存在的被修复/未修复的漏洞所持续的时间：

![](/images/2019-01-16/assets/1539082174114.png)

论文附录中还有更为详尽的测试数据，本篇论文绝大部分篇幅为各种样本的测试数据和测试结果分析。

#6 A GENERIC ANDROID DOS ATTACK

作者发现了一种新方法，可以通过使Android OS中的system_server进程遇到OutOfMemoryError状态来触发Android上的受控引导循环攻击，从而导致系统重启。这是通过零权限app重复使用特定的API方法调用来完成的，其中方法调用的参数最终将存储在system_server进程的堆上，在Android中启动进程（包括system_server）时，会为其分配固定的最大堆大小。一旦system_server分配了它的所有堆内存，如果不能释放任何内存，那么最后会crash。

app还可以通过提供从BroadcastReceiver继承的对象和包含一个或多个操作字符串的IntentFilter对象来动态注册广播接收器。 system_server管理ActivityManagerService类中的所有app组件。当在广播接收器注册期间提供操作字符串时，它存储在可以保存任意数量的数据的变量中（在IntentResolver类中名为mfilters的HashSet变量）。因此app可以提供大超长的字符串以存储在system_server进程的堆上以耗尽其内存，从而导致crash。

# 7 CONCLUSION

作者设计并实现了Daze用于fuzz Android应用和Android操作系统中的异常。作者发现了超过50％的当前Android设备容易受到系统DoS攻击，这是由Android App和系统代码中的异常处理不足所致。 作者还分析量化了连续版本的应用中致命异常的暴露期。结果显示，应用程序中的大多数致命异常都是从之前的应用版本继承而来的。此外作者还发现系统崩溃导致某些流行的Android设备上的数据泄漏漏洞。最后提出了一种通过使用来自零权限app标准API耗尽堆内存触发Android设备DoS的攻击方式。
