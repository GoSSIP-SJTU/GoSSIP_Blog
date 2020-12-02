---
layout: post
title: "iDEA: Static Analysis on the Security of Apple Kernel Drivers"
date: 2020-11-24 15:43:09 +0800
comments: true
categories: 
---

> 作者：Xiaolong Bai, Luyi Xing, Min Zheng, Fuping Qu
> 
> 单位：Orion Security Lab, Alibaba Group. Indiana University Bloomington
> 
> 会议：CCS 2020
> 
> 论文链接：[iDEA: Static Analysis on the Security of Apple Kernel Drivers](http://homes.sice.indiana.edu/luyixing/bib/CCS20-iDEA.pdf)

## Abstract

Apple OS（例如iOS，tvOS，iPadOS，macOS等）的driver在kernel space中运行，驱动漏洞可能招致严重的安全后果。尽管驱动漏洞比例很高，但从未有工作对Apple驱动进行系统的静态分析与漏洞检测。作者开发了第一个自动静态分析工具iDEA，用于查找Apple driver binary中的bug，并针对两种常见的安全问题（条件竞争和越界读/写）进行检测，对主流Apple OS的15个版本的共3400个Apple driver binary进行首次大规模研究，发现了35个0day。

<!-- more -->

## 1 Introduction to Apple Driver

### 1.1 Driver Programming Model

Apple的Driver Programming Model大致需要了解以下几点：

* 每个驱动都被实现为C++的driver class
* driver class必须继承自kernel class IOService并且实现一套虚函数(由内核调用，称为driver-callback)。例如内核实例化驱动后，在驱动实例上调用`start()`来启动驱动程序。

    | Callback Name | Description |
    | :---: | :------: |
    | start() | the driver is about to start | 
    | stop() | the driver is about to stop | 
    | initWithTask() | the driver is about to be initialized | 
    | setProperties() | set the properties of the driver |
    | newUserClient() | create the driver’s UserClient instance |

* 不允许driver class使用自定义的构造函数/析构函数，要求采用预定义的marcos `OSDefineDefaultStructors`来生成构造函数/析构函数。如果驱动的代码空间中没有显式调用，则构造函数将从驱动程序二进制文件中删除。
* 已安装的驱动程序二进制文件及其配置文件可以在Apple OS的`/ System / Library / Extensions`下找到，以`AppleKeyStore.kext`为例，其文件组成为：

![](/images/2020-11-24/IOSurface.png)

这些driver binary一般都很小，通常为几百KB。

![](/images/2020-11-24/size.png)

### 1.2 Registering Drivers to Apple Kernel

为了访问硬件设备，内核首先确定要使用的驱动（基于硬件类型），然后利用driver instance来操作设备（通过调用实例方法）。**这要求所有的driver都注册到kernel**。

当driver binary加载到内核时，内核会实例化driver instance，并将所有instance维护在runtime pool中以备使用，后续可以通过类名称实例化driver（调用`OSMetaClass :: allocClassWithName()`内核API）。

注册类信息是通过InitFunc完成的。Apple driver compiler在binary中放置了一个特殊的段`__mod_init_funcs`，包含了初始化函数的函数指针表InitFuncs，每个InitFunc对应于binary中的一个特定的driver class。

![](/images/2020-11-24/segments.png)
![](/images/2020-11-24/initfuncs.png)
![](/images/2020-11-24/init.png)
![](/images/2020-11-24/metafunc1.png)

在运行时加载binary时，内核会自动执行`InitFunc`从而将driver类信息注册到kernel。每个InitFunc都将driver class的信息包装到MetaClass(从内核类OSMetaClass继承的子类)的对象中。

<!-- ![](/images/2020-11-24/f10.png) -->
![](/images/2020-11-24/f1.png)
<!-- ![](/images/2020-11-24/meta.png) -->

要创建某个驱动的实例时，API会从映射中查找相应的MetaClass对象，并调用其`MetaClass :: alloc()`，从而创建driver实例。该函数会根据driver class的大小为该实例分配内存，并为该实例分配一个vtable。

![](/images/2020-11-24/metafunc2.png)

在运行时，内核会为注册到内核的所有驱动程序维护MetaClass对象的映射。这些对象使Apple内核可以通过`allocClassWithName()`轻松实例化驱动程序（类）实例。

### 1.3 Interacting with User-space Programs

为了服务用户空间的请求，driver class通常具有一个名为`UserClient`的伴随类，使用delegate机制暴露给用户空间。

![](/images/2020-11-24/f2.png)

首先，（用户空间）程序通过系统API调用`IOServiceGetMatchingService`获取需要的driver instance的handle。对应地，内核从其runtime信息池中查找driver instance。

然后，用户空间程序使用driver的handle通过系统API调用`IOServiceOpen`来获取driver对应的UserClient实例的handle。对应地，内核在驱动的实例上调用`newUserClient`实例化UserClient对象.

要使用UserClient handle访问驱动，用户空间程序可以通过特定的系统API触发user-entry。例如从用户空间程序调用系统API`IOConnectCallMethod`后，内核将在UserClient实例上调用`externalMethod`。

![](/images/2020-11-24/externalmethod.png)

## 2 Challenges

针对苹果操作系统的驱动的安全分析，与Linux等操作系统不同，存在一些独特的挑战，主要是由于苹果独特的driver programming model、driver registration以及interface management产生的。

* 尽管苹果的驱动也是C++代码编写的，但符号都被strip掉了，constructor通常也会被移除。另外其运行时动态分配的特性更是加大了恢复C++ class的难度。
* 苹果驱动的接口管理也不像Windows、Linux那么友好，导致要系统地找到驱动的entry points较为困难。
  
## 3 Design

作者设计了iDEA这一针对苹果驱动的二进制代码的静态分析工具，整体流程如图3所示:

* Phase1-2分别是针对前面提及的两个挑战，即C++ class的恢复和程序entry point的定位。
* phase3-4也是程序分析中的常见问题，借助类型推断来识别对象以及解决间接调用。
* 在phase1-4的基础上，作者构建了以entry point为起点的过程间控制流图（ICFG）。基于ICFG，可以在iDEA这个框架上搭建policy engine进行漏洞检测。作者使用了污点分析和符号执行，分别来检测条件竞争（Race condition）和越界读写（out-of-bound read/write）的漏洞。(本质上还是内存破坏漏洞)

![](/images/2020-11-24/design.png)

### 3.1 Phase I: Recovering Driver Classes

Phase1 旨在从driver binary中恢复所有的class，包括vtables和inheritance hierarchies。Insight很直接：就是利用driver class注册到kernel的机制，遵循kernel收集driver class信息的过程。

> 找到`_mod_init_func`段-->找到目标driver class的InitFunc

分析InitFunc：
![](/images/2020-11-24/init_insn.png)
* **查找driver class的name和size：** 从作为参数的寄存器里取得。实现上来说，先做后向切片把影响这些寄存器的指令切出来，然后使用前向传播来识别这些值。
* **进行前向分析来查找driver class的vtable：**
  * 依据：MetaClass对象的虚函数`xxx::MetaClass::alloc()`中将driver class的vtable赋值给driver instance。
  ![](/images/2020-11-24/alloc_insn.png)
  * 方法：从`InitFunc`中找到MetaClass的vtable地址，从而在其vtable中找到MetaClass::alloc函数，再去alloc函数中找到赋值给driver class的vtable。
* **恢复driver class hierarchies：**
  * 依据：OSMetaClass函数原型`OSMetaClass::OSMetaClass(OSMetaClass *__hidden this, const char *className, const OSMetaClass *superclass, unsigned int classSize)`，其中有一个参数是super class，能够反映继承关系
  * 方法：从`_mod_init_func`中遍历所有对象的InitFunc，找其中调用OSMetaClass中传入的superclass参数，就能找到继承关系。

### 3.2 Phase II: Discovering Driver Entry Points

UserClient:
* inherits from a generic class IOUserClient
* implements a set of virtual functions inherited

Entry points是Apple driver暴露给用户空间的interface。作者总结了两类：

#### Type-1: UserClient的虚函数

UserClients继承自IOUserClient，且Apple在C ++中要求单一继承（意味着父类和子类的虚函数的偏移量是相同的），因此可以直接比较UserClient和IOUserClient 的vtable，那些覆写的函数指针就是这类user-entry。

![](/images/2020-11-24/f4.png)

一些未被UserClient覆写的函数指针是需要过滤的，因为这些会流入父类的通用实现中，这些实现：（1）在内核中–在驱动程序分析的代码空间中不可见；（2）无效-通用实现未定义要对特定子类执行的操作。

#### Type-2: UserClient-internal functions

苹果在一类称为method-struct的数据结构中管理这类entry的函数指针和其他信息（如input buffer）。UserClient的所有方法结构在driver binary的data部分中形成一个数组，每次entry会返回从这个数组中返回一个method struct。

作者的方法是检查这些user-entry来找到对数组的引用，从而找到其记录的所有函数指针：

* 找到返回method-struct的指令，关注RAX/X0寄存器
* 后向切片找到影响这些寄存器的指令
* 找到其中的address-loading指令，这些指令从data段load method-struct的地址，就是要找的entry。

| Name | Description | Corresponding system APIs
| :---: | :------: | :------: |
| externalMethod() | provide methods to user-space programs | IOConnectCallMethod
| getTargetAndMethodForIndex()  | provide methods to user-space programs (legacy user-entry)| IOConnectCallMethod
| getAsyncTargetAndMethodForIndex() | provide methods that return results asynchronously (legacy user-entry) | IOConnectCallAsyncMethod 
| getTargetAndTrapForIndex() | similar to getTargetAndMethodForIndex (legacy user-entry) |IOConnectTrapX
| clientMemoryForType() | share memory with user-space programs | IOConnectMapMemory
| registerNotificationPort() | allow user-space programs to register for notifications| IOConnectSetNotificationPort
| setProperty() | set runtime property of the userclient|IOConnectSetCFProperty｜
| clientClose() | stop using the userclient|IOConnectSetCFProperty

### 3.3 Phase III: Identifying Objects with Vtables

从driver entry point开始构建的ICFG，会遇到很多虚函数调用。这种虚函数在对象上的调用是通过间接调用进行的。因此需要识别出赋给对象的vtable来解决。

<!-- ![](/images/2020-11-24/f5.png) -->

之前的方法都是依赖于对constructor进行分析，找一些特定pattern的vtable assignment。但是在苹果平台不适用，因为这些driver class通常是通过kernel api进行实例化的，没有constructor，而且还需要runtime特性分析。作者是进行类型推断，利用Phase I识别的vtable-class对应关系，分析kernel API `OSMetaClass::allocClassWithName`。

#### Type Infer

作者总结了三类情况去识别对象的类型：

* 在driver的函数本地构造的对象
  * 通过classname实例化的对象：`allocClassWithName(char * classname)`。后向切片+前向常量传播 --> 找到最终参数的string地址，这个classname能反映object的类型。
  * 通过构造函数实例化的对象：driver的代码pattern是比较特别的，可以比较方便地找到constructor。
* 作为参数传给driver函数的对象
  * 作为参数传给UserClient的driver class对象（`newUserClient() driver-callback`)：
    * 配置信息完整，则利用Info.plist文件中的键值对信息
    * 否则，利用如下的代码信息，这种动态地进行类型强制转换的函数中有我们需要的target type，可以通过后向切片+前向常量传播来找到。
    `bool IOReportUserClient::start(IOReportUserClient ∗this, IOService ∗a2) {v2 = OSMetaClasBase::safeMetaCast(a2, IOReportHub::MetaClassObj);`
  * 作为参数传递给driver-callback的内核对象：
    * 这种driver callback是从IOservice继承的虚函数，定义在内核框架I/O Kit，是有符号的，于是就可以比分方便的推知类型，例如`__ZN9IOService4initEP12OSDictionary`
* 由kernel API返回的对象:
  * 对象是kernel API调用的参数或返回值：参考Apple API文档或是头文件。

#### Type propagation

逐个基本块级地顺着控制流进行传播。传播过程中维护一个object-type map进行更新。过程中需要与下一部分的Resolving Indirect Calls交替进行。

### 3.4 Phase IV: Resolving Indirect Calls

利用先前阶段的输出（已恢复的类，vtable和对象类型），可以解析控制流中的间接调用并生成ICFG。考虑到多态性，调用图中的某些节点可能具有多个边。

作者还观察到了另一种Apple driver的的间接调用方案可以处理，以使ICFG更加完整：driver class将其函数指针传递给内核，然后由内核通过函数指针进行间接调用。当iDEA在driver代码中找到此类内核API调用时，将执行向后切片以跟踪传递给API的函数指针。然后，iDEA选择不直接将调用图的边缘从调用位置扩展到函数指针的目标，而是将其扩展到内核代码空间中。

第三阶段和第四阶段反复进行。识别对象的类型后，阶段IV解决对该对象的间接调用。确定调用目标后，可以分析和识别更多对象的类型。

### 3.5 Supporting Pluggable Policy Checkers

Checker被实现为具有两个callback函数的插件：`pre_instr_checker`和`post_instr_checker`，分别在遍历ICFG时解析每一条指令的前、后调用。

iDEA实现了两个analysis client，分别是taint analysis client（TAC）和symbolic execution client（SEC）：

* TAC基于Triton（dynamic taint tracking tool），在register/memory area level，通过模拟执行来进行。
* SEC也是依赖于Triton，借助其TritonContext模块来赋予符号值。

## 4 Security Policy Checker

### 4.1 Race Condition Checker

Apple driver management隐式地将任何driver/UserClient实例视为共享资源而不进行保护。那么用户空间不同的线程就能够使用同一个UserClient handle去触发同一个UserClient的entry，这种理应加上lock保护，不然会引入条件竞争漏洞。

对于条件竞争的checker设计，作者主要针对会造成UAF和空指针解引用的情况进行考虑，采用了下图所示的两条策略进行匹配。为了提高准确率，还设计了几条补充规则来进一步限制。

![](/images/2020-11-24/race.jpeg)

### 4.2 OOB Read/Write Checker

对于OOB R/W，作者定义的漏洞模式是：驱动使用来自用户空间的输入作为索引来访问内核空间的buffer，且该输入未进行boundary check。于是，对于前半句话，作者使用污点分析，对于后半句话，作者使用符号执行。


## 5 Evaluation

在实验评估过程，作者的针对多个苹果操作系统的共3400个驱动程序进行分析，而且其中版本都非常新，例如iOS 13.6、macOS 10.15.6等。

结果非常亮眼，共发现了46个unique bug，其中包含了13个UAF、28个Null pointer dereference以及5个OOB bug。这些bug中，35个0day漏洞（5个获取了CVE），另外11个是较老的OS版本中的问题，已经在新版本中修复了。

除了漏洞方面的成果外，作者研究的类恢复、entry point定位以及间接调用的识别，也取得了不错的效果：

* **Recovering classes：** 针对362个driver binary种的总共8217个classe(包括3841个由driver compiler生成的MetaClasses) ，iDEA 恢复了7,666 (93%)个classes, 包括他们的vtables、inheritance hierarchies、class names和class sizes。iDEA不能恢复的7％是driver开发人员实现的utility class（不是源于Apple框架，也就没有InitFuncs）。
* **Discovering driver entry points：** 针对macOS、iOS、iPadOS和tvOS，iDEA分别发现了1426、348、371和310个entry points。对四个由源码的driver，与源码提供的ground truth比较，iDEA以100%的准确率找到了每个平台的所有的54个entry points。
* **Resolving indirect calls：** 在macOS上的362个driver上，iDEA 解决了139282/209558个 indirect calls (66%)。在iOS、iPadOS、tvOS上，iDEA resolved 63% (58163/92464), 63% (66861/105950), 59% (43621/74768) of indirect calls。

发现的bug列在下面的表中，这几个cve也都是很强的，能实现任意代码执行。

![](/images/2020-11-24/t6.png)

作者也有一些评估的数据，如下：

![](/images/2020-11-24/t3.png)

## 6 Limitation

主要有三点局限：

* 没能实现跨结构的分析，作者对x86_64和arm64的分析是分别实现的。
* 对Apple runtime feature没有完全解决。
* checker还是比较简单，存在一定误报。

实际上工作的难度，并不在后续check的设计，而在前面4个phase的研究上，需要相当丰富的apple OS研究经验才能够找到解决的方法。另外这种kernel space的内存破坏漏洞的利用也是个难题。