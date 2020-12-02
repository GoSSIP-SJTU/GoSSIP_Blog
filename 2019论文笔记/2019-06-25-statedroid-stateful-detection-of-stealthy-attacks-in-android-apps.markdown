---
layout: post
title: "StateDroid: Stateful detection of stealthy attacks in Android apps"
date: 2019-06-25 16:55:54 +0800
comments: true
categories: 
---

作者：Mohsin Junaid, Jiang Ming, David Kung

单位：University of Texas at Arlington

出处：Annual Computer Security Applications Conference, ACSAC 2018

链接：[Paper](https://dl.acm.org/citation.cfm?doid=3274694.3274707), [Website](http://www.apkscanner.com/index.jsp), [Github](https://github.com/mohsinjuni/statedroid)

<hr/>

## Introduction

越来越多的Android恶意软件通过延长生命周期来隐藏自身的恶意行为，这类恶意软件相当长的一段时间内都没有被发现。作者根据对这类软件的分析经验总结出有以下三个特点：

1. 隐蔽攻击经历多个状态
2. 状态之间的转换是由一系列攻击动作引起的
3. 攻击动作通常涉及不同对象上的多个Android API

于是作者设计一个名为StateDroid的双层有限状态机（FSM）模型，用于描述状态转换方面的多步骤隐蔽攻击。StateDroid把Android API和攻击的语义抽象为霍恩子句，然后通过霍恩子句验证自动构建双层FSM模型。作者开发并开源了StateDroid的原型，使用1,505个Google Play应用程序和1,369个恶意应用程序进行了评估，并发现7.5％的Google Play应用程序存在了隐蔽攻击。
<!--more-->
## Background

Stealthy Attacks：执行其他操作以隐藏其恶意行为的攻击，例如在后台阻止，接听或拨打电话，泄露敏感信息（例如短信和通话记录），以及录制音频或视频文件。

Action：执行一些应用程序特定的功能

Object State：Object的某个时间点确定的内部数据或属性值

### Motivating Example

![image-20190624184206645](/images/2019-06-25/assets/image-20190624184206645.png)



这是一个来自Android.HeHe真实的恶意软件样本，此攻击会自动阻止来电，并通过静音设备振铃来隐藏恶意行为。在接收到来电时，会以特定顺序执行三个动作：

1. 将设备振铃器设置为静音模式; 

2. 阻止或断开来电; 

3. 将设备振铃器设置回正常模式。 

第一个和最后一个操作在按给定顺序执行时，会隐藏用户的恶意行为。为拦截来电，此恶意软件会注册广播接收方`IncomeCallAndSmsReceiver`。收到呼叫后，系统会调用其onReceive（）生命周期回调并执行攻击代码。第5-6行可以访问音频管理服务并将设备振铃器设置为静音模式，然后使用第10-14行的Java反射阻止电话呼叫。最后，第20行将振铃器恢复到正常模式。

作者认为先前的检测模型不足以捕捉到一个攻击涉及多个状态的复杂隐身攻击。传统的控制流/数据流模型可能会错过攻击。于是作者提出了一种双层有限状态机（FSM）模型来自动检测隐身行为。因为通过以特定顺序执行一组动作来完成隐身攻击，因此FSM模型可以恰当地描述攻击动作之间的时间顺序。

FSM表示为图形，其中节点是状态，edge表示状态转换。某种特定的攻击由FSM表示，称为攻击状态机（ASM），其中状态是根据许多已执行的操作描述攻击的状态。两个状态之间的过渡代表了所涉及的操作。 当ASM中的最后一个操作被执行的时候，这个攻击会被StateDroid认为存在。


在检测攻击操作时会有一些难点，因为每个操作的实现都受到许多因素的影响，例如对多个对象的API调用，API调用序列的顺序以及特定的输入值。此外，对象也可以向下传递给多个类或方法，并且跟踪对象状态也非常重要。它要求根据调用它们的API正确更新对象状态。

为了解决这些问题，作者引入了另一层FSM，即对象状态机（OSM），以跟踪对象状态。为每个对象维护OSM，并根据感兴趣的调用API进行更新。如果API调用完成了操作功能，OSM将报告操作的检测。基于模型检验的安全性分析的共同局限是很难自动构建高级检测模型。作者将Android API语义，霍恩子句。然后可以通过霍恩子句验证自动生成ASM或OSM。



## STATEDROID OVERVIEW

![image-20190624185747259](/images/2019-06-25/assets/image-20190624185747259.png)

为了检测类似实例中的隐蔽攻击，StateDroid一次执行三项工作：

- 分析onReceive()方法的字节码，以识别三种攻击行为中涉及的`API`（静音设备铃声，阻止来电， 并恢复设备铃声）;
- 根据识别的`API`，输入参数值和所涉及`Object State`检测动作; 
- 使用检测到的`Action`的顺序来检测攻击。

这些任务分别由StateDroid的三个组件完成：

### API调用检测器

首先从组件生命周期中获取回调序列，对序列中的每个回调函数进行静态检查，以识别可用于发起攻击的Android API调用。例如标识实例中第5-6行和第10-14行调用的API调用。

StateDroid调用Androguard来生成控制流图（CFG），在CFG上执行遍历以产生用于分析基本块的部分顺序。通过这种方式，CFG中的每个前基本块在其后继基本块之前被分析。 StateDroid使用一堆查找表来维护有关指令创建，更新或使用的对象和基元的信息。表的条目存储所有对象的数据值，数据类型和标识符（或名称）等信息。使用全局查找表维护全局级对象和原语（如静态字段）。API调用事件包含API调用，接收器对象，参数和调用上下文的信息。

进入第2行的onReceive（上下文contxt，Intent）方法时，会创建一个查找表并将其推送到查找表中。并在查找表中为传入的两个参数创建了两个条目作为输入参数。分析检测第5行的`getSystemService("audio") `API调用，API调用检测器为其生成API调用事件并将其发送给Action Detector。同样，对于第6行的`setRingerMode(0)`API调用，创建API调用事件并将其发送到Action Detector。

### 操作检测器

分析生成的API调用序列和所涉及的对象，例如使振铃器静音并阻止来电。由于攻击操作可能涉及多个对象，并且API调用可能会更改被调用者对象的状态，因此作者设计了对象状态机（OSM）来模拟攻击操作行为。 API调用序列用于触发OSM的状态转换。每当OSM到达指示操作完成的状态时，就会检测到攻击操作并将其广播到下一个组件。

将API调用事件作为输入，动作检测器识别可用于发起攻击的动作。动作执行app level功能，例如将铃声设置为静音模式，开始呼叫和阻止呼叫。操作通常涉及对多个Android对象的多个API调用。例如，将振铃器设置为静音模式需要两个API调用：

（1）通过调用Context对象的getSystemService(…) 函数获取AudioManager对象

（2）调用AudioManager.setRingerMode(0) 来设置振铃到静音模式。

作者认为第二个API调用不需要严格地在第一个API调用之后发生，只要第二个API调用发生在第一个之后。而且，完成动作的API调用序列不是唯一的。例如，以下操作序列还将振铃器设置为静音模式：

（1）从应用程序的上下文中获取AudioManager对象

（2）调用此对象的setRingerMode(2) 以将振铃器设置为正常模式

（3）调用对象的setRingerMode(0) 将振铃器设置为静音模式。

![image-20190624205130583](/images/2019-06-25/assets/image-20190624205130583.png)

 图3显示了两个用于检测ringer_silent和ringer_normal操作的OSM。 最初输入的是上下文对象，如状态S0所示。 每当接收到对上下文对象的`getSystemService("audio")` API调用事件时，就会发生状态转换。 结果，进入第二个OSM的状态S1。 在此之后，许多API调用事件可能会到达，但只有setRingerMode（0）或setRingerMode（2）事件可以导致第二个OSM进行状态转换并将ringer_silent或ringer_normal操作的广播。

### 攻击检测器

StateDroid使用攻击状态机（ASM）对每类攻击建模。 ASM的每个状态表示攻击的状态，而转换表示检测到的攻击动作。默认情况下，每个ASM都以初始状态开头。如果检测到攻击操作则会转换到新状态，并且ASM处于期望攻击操作的状态。如果其中一个ASM达到攻击状态，则识别输出潜在攻击。

![image-20190624205822304](/images/2019-06-25/assets/image-20190624205822304.png)

 图4显示了三种攻击状态机，用于在三种不同情况下检测Call Blocked攻击。 ASM的每个状态表示攻击的状态，而转换表示检测到的动作。 每个ASM以初始状态开始（由带有点尾的小箭头指向），并以攻击状态结束（由灰色圆圈表示）。 如果从动作检测器接收到动作，并且ASM处于期望动作的状态，则它转换到新状态。 如果其中一个ASM进入攻击状态，则会检测到攻击。 为了构造ASM，作者使用霍恩子句来指定动作的效果以及攻击的敌对意图。 执行定理证明以生成动作序列，每个动作序列以攻击的初始状态开始并进入攻击状态。 然后使用动作序列来生成攻击状态机。

霍恩子句会指定攻击对初始状态和攻击状态的影响。 例如，Call Blocked攻击会结束呼叫。 它涉及两个对象：Telephony对象和AudioManager对象，这两个对象必须存在于"Ringer Normal"初始状态。 只要Telephony对象的状态是“呼叫结束......”，就会检测到Call Blocked攻击。

## EVALUATION

StateDroid首先对16个手工制作的应用程序进行分析，API Call Detector为其生成23个API调用事件，作者确认Action Detector成功检测到这16个手工制作应用中的所有操作。

对恶意软件应用程序的评估：来自Genome和Contagio数据集的1275个样本的分析结果中，StateDroid检测到代表22个操作类别的604个操作。构成检测到的动作的62％（373）的前五个动作是信息泄露（161），发送SMS消息（81），中止SMS或呼叫通知（56），从SMS和呼叫内容提供者（43）删除数据，以及打电话（32）。作者对604个检测到的操作执行手动验证，并发现没有检测到的操作是误报。图5显示了检测出的恶意行为最多的前25个恶意软件家族。其中info_leak是最常见的操作，14个恶意软件系列表现出5个或更多操作。

![image-20190624213902332](/images/2019-06-25/assets/image-20190624213902332.png)



图6显示了作者收集的所有7个勒索软件系列的16个检测到的操作类别。 虽然图6和图5中的许多操作都是重叠的，但Action Detector还会检测勒索软件系列中的两个唯一操作（lock_device，encrypt_file）。 图6显示5个系列使用`lockNow()`API锁定设备，锁定后，其中3个系列也使用更改设备密码。

![image-20190624214238493](/images/2019-06-25/assets/image-20190624214238493.png)



与本文工作相关的工具是AsDroid(Anti-Stealth Droid), AsDroid通过检测用户界面组件的代码行为与该组件上显示的文本之间的不匹配或不兼容来检测特殊类型的隐身行为。 作者从每个恶意软件系列中随机选择了49个样本。由于AsDroid源代码不可用，所以作者模拟了49个样本的AsDroid分析，比较如下。:

- AsDroid检测到4个Action，而StateDroid可以检测另外10个其他Action。 
- AsDroid检测7个样本中的SendSMS动作，而StateDroid报告了8个。 
- Statedroid和AsDroid在两个样本中都检测到了PhoneCall动作。 
- StateDroid报告36个样本中的敏感信息泄漏，而AsDroid报告43个。 

此外，作者还把StateDroid与FlowDroid和Dexteroid与这49个样本进行比较。 FlowDroid，Dexteroid和StateDroid分别报告28、112和114次警告。作者手动验证所有FlowDroid警告都是TP，并且是Dexteroid 的112个警告的一部分。然而，在Dexteroid输出中发现了11个误报，留下了101个TP。 StateDroid发现了六种新攻击，而Dexteroid和FlowDroid没有发现任何这些新发现的警告。作者认为因为FlowDroid和Dexteroid是数据流/控制流分析，所以仅限于检测信息泄漏，它们无法检测到StateDroid可以检测到的许多其他隐蔽攻击。

所有的实验都在AMD Phonom II四核处理器上运行，运行Ubuntu 14.04操作系统，内存为16 GB。 StateDroid完成应用程序分析的最长时间为1,770秒，最短时间为120秒, 平均需要214秒。 API Call Detector消耗大部分时间（195秒），包括将应用程序反汇编为字节码，导出回调序列，检索每种方法， 分析字节码指令，使用查找表条目存储和检索对象，检测API调用以及向Action Detector发送API调用事件。