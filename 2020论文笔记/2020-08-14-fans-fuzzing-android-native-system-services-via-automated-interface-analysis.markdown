---
layout: post
title: "FANS: Fuzzing Android Native System Services via Automated Interface Analysis"
date: 2020-08-14 11:08:35 +0800
comments: true
categories: 
---

> 作者：Baozheng Liu1,2∗ , Chao Zhang1,2, Guang Gong3 , Yishun Zeng1,2 , Haifeng Ruan4 , Jianwei Zhuge1,2
> 
> 单位：1Institute of Network Science and Cyberspace, Tsinghua University; 2Beijing National Research Center for Information Science and Technology; 3Alpha Lab, 360 Internet Security Center; 4Department of Computer Science and Technology, Tsinghua University
> 
> 会议：USENIX Sec' 20
> 
> 链接：[FANS: Fuzzing Android Native System Services via Automated Interface Analysis](https://www.usenix.org/system/files/sec20fall_liu_prepub.pdf)

## 简介

作者提出了一个自动的基于生成的fuzz Android native系统服务的方法，之前很少有工作关注这个方面，这其中存在3个挑战：

1. multi-level interface识别。只有top-level interface会在Service Manager中注册，有很多嵌套的interface可与被用户app通过top-level interface调用。
2. 提取interface模型。为了有效的生成transaction的输入data，需要提取目标interface语法。不同的transaction可能有不同的语法，语法可能还与路径要求共存，如分支条件，循环，嵌套循环等
3. 生成语义正确的输入。需要通过各种各样的检查，如对size的检查。

FANS在6个Android版本为android-9.0.0_r46的设备上运行了30天，从上千个crash中发现了30个不同的漏洞和触发了138个不同的Java异常。

<!-- more -->

## Background

### Android系统服务

Android的系统服务分为两类：1）Java系统服务，主要使用JAVA实现 2）native系统服务，主要用C++实现。

native系统服务既可以调用java code也可以被调用。

从Android8以后，所有服务又被划分为3个domain，normal domain，vendor domain和hardware domain。normal domain直接位于 Android Open Source Project (AOSP)里，其他两个取决于厂商和硬件。

此外，并非所有接口都是用C ++静态定义的，其中一些接口是用Android Interface Definition Language（AIDL）定义的。 构建Android映像时，将调用AIDL工具来动态生成适当的C ++代码以进行进一步编译。

**应用-service通信模型**

service先要在service manager上注册，然后监听和处理应用的消息。应用需要向service manager查询接口（封装在代理Binder对象），得到一个top-level 接口。然后它可以通过这个接口去获取multi-level接口，调用相关的transaction。

![img](/images/2020-08-14/fig1.png)

**Android系统服务接口**

App通过RPC接口transact()调用目标transaction。

```java
IBinder::transact(code, data, reply, flags)
```

service端有个分发器负责通过code处理这个请求

```java
onTransact(code, data, reply, flags)
```

通常，每个服务都有一组可以通过RPC(Remote Procedure Call)调用的方法。 它们在基类中声明，但分别在客户端代理和服务器端stub中实现。 

这个机制也应用于multi-level接口，但是它们对应的binder不会在service manager中注册，只能由top-level接口调用。

## Design

设计思路：

1. 以RPC为中心的测试。如果直接构造transaction的事件，容易产生很多误报，因为攻击者并不能构造任意的事件，只能通过IPC机制（binder）生成有限的事件。
2. 基于生成的fuzzing。基于mutation的fuzz会产生大量不合法的输入格式和语义，不能正确的处理或者反序列化。
3. 从代码中学习输入模型。作者注意到输入模型知识藏在源代码中，选择分析Android源代码以自动检索输入模型。

### overview

interface collector 收集目标service中所有的接口。

interface model extractor提取输入输出的格式和变量的语义（变量名，类型，结构/枚举/类型别名的定义）。

denpendency inferer推测接口的依赖，transaction间/内变量的依赖。

fuzzer engine随机生成transaction和调用接口。同时设计了一个manager负责同步host和手机之间同步数据。

![img](/images/2020-08-14/fig2.png)

### Interface Collector

作者检查在AOSP编译命令中作为源出现的每个C/C++文件，以便可以收集AIDL工具在编译过程中动态生成的接口，而不是直接扫描C/C++文件的`onTransact`方法。

### Interface Model Extractor

1. 从服务器端提取代码。因为服务器端的接口代码较为集中，并且可以看到service如何就从data反序列化出输入，以及输出如何序列化到reply。

2. 将所有AIDL转化为C++文件（反之会丢失信息）一起处理，生成抽象语法树（AST）。

3. 识别transaction code。在目标interface中，OnTransact()函数通过不同的code使用switch语句进行分发到目标transaction，在AST中被转化为多case节点，通过分析case节点识别相关的固定code。

4. 提取输入输出变量。输入变量会从data包中反序列化出来，输出变量会序列化到reply。有三种可能的变量类型：

   - 顺序变量：没有先决条件。对应在sequential statement中读入的变量。

     主要会使用7种语句：`checkInterface`，`readXXX`，`read(a, sizeof(a)*num)` , `read(a)` , `readFromParcel`, `callLocal`,Misc Function。

   ![image-20200706182957781](/images/2020-08-14/list1.png)

   ​	

   - 有条件变量：如果条件不满足可能会是`NULL`或者不出现在数据中，也可能会有不同的类型。`fd`是条件变量。

   ![image-20200706183038431](/images/2020-08-14/list2.png)

   - 循环变量：在循环或者嵌套循环中反序列化。`key`, `fd`和`value`都被认为是循环变量。

   ![image-20200706183109556](/images/2020-08-14/list3.png)

   还有一种比较特殊的是return语句的返回值，如果条件不满足会返回error code。这里会尽量生成不超过`MAX_BINGDER_TRANSACTION_SIZE`的值，使其不会触发返回error code的return语句。

   ![image-20200706184156968](/images/2020-08-14/list4.png)

5. 提取类型定义

   ​	1）结构类型。直接从AST提取。

   ​	2）枚举定义。提取给定的常量值。

   ​	3）类型别名。识别typedef语句。

   ![image-20200706184646160](/images/2020-08-14/list5.png)

### Dependency Inferer

#### Interface Dependency

1. 生成依赖。如果一个interface是通过另一个interface获取的，那么它们两之间就有生成依赖。upper-level interface调用`writeStrongBinder`序列化一个深层的interface到reply。

2. 使用依赖。如果interface A被另一个interfaceB使用，B调用`readStrongBinder`从data包中反序了化A。

#### Variable Dependency

1. Transaction内的依赖。包括条件依赖，循环依赖，数组大小依赖。

2. Transaction间依赖。判断有4条标准：1）一个是输入变量，一个是输出变量；2）两个变量在不同的transaction里；3）变量类型相同；4）非基本类型，或者输入和输出的变量名相似。

   ![img](/images/2020-08-14/algo1.png)

### Fuzzer引擎

**Transaction Generator** 生成transaciton的输入依次遵循下方三个原则：

1. 约束。生成变量时要确认约束。
2. 依赖。如果一个transaction是由另一个transaction生成的，大部分情况下作者都会先生成被依赖的transaction，然后获取其输出的reply。
3. 类型和名称。通过名称和类型生成相应的值，比如给`opPackageName`生成一个合法的包名，给`pid`生成int值。

**Interface Acquisition** 通过service manager获取top-level interface，通过interface依赖去fuzz multi-level interface。



## Evalutation

实验环境：

前三个组件实现在Ubuntu 18.04 with i9-9900K CPU, 32 GB memory, 2.5 T SSD.

测试手机为Pixel * 1, Pixel 2XL * 4, 和 Pixel 3XL * 1，由于不同机型的source code有少许不同，前两个都是在Pixel 2XL设备测试。

固件版本PQ3A.190801.002, 也就是 android-9.0.0_r46

### interface统计和依赖

interface的统计结果如图3所示，存在37%的multi-level interface，这些是不在service manager里的，说明了作者这种提取方式的重要性。

![img](/images/2020-08-14/fig3.png)

这里展示了部分的interface依赖图如图4，一个multi-level interface可以送不同的upper interfac获取，也就是说可以从不同路径去fuzz同一个interface。

![img](/images/2020-08-14/fig4.png)

### 提取的interface模型

左边是transaction的数量，右边是transaction path的数量，transaction path是transaction数量的1.5倍，说明很多transaction都有一个以上的return。

![img](/images/2020-08-14/fig5.png)

从左至右依次是变量的模式，别名，依赖的统计情况。

![image-20200706142625826](/images/2020-08-14/fig6.png)

### 漏洞发现

发现的漏洞如表2所示，包括一些从library和linux系统组件中出现的漏洞，说明FANS生成的输入可以在复杂的约束下将控制流驱动到更深的路径中。向google报告后收到了20个确认回复。

![image-20200706144648381](/images/2020-08-14/tab2.png)

### Case Study

#### I: new_capacity overflow Inside readVector of IDrm
在IDrm中由多个`new_capacity`的漏洞，都是由同一个函数`BnDrm::readVector`触发的，它调用了函数`insertAt`会开辟一个缓存区，大小由data里的size决定。在`insertAt`里有对`index`的检查但是没有对`size`的检查。当

#### II: Out-of-bound Access Inside informAllUidData of statsd
`statsd`是Android9的一个守护进程。在transaction `Call::INFORMALLUIDDATA`中，`statsd`数据包中反序列化三个vector，分别包含int32_t，int64_t和::android::String16的项。这些vector将传递到`notifyAllUidData`中，然后转发到`UidMap`的`updateMap`方法。函数`updateMap`循环访问三个vector。其中，`uid`的大小用作循环计数。由于任何一个vector中的item都应该与其他两个vector对应，因此这三个vector的长度应相同，但是在函数体内没有进行验证。可以通过传入比其他两个更长的uid vector来实现越界访问。

#### III：Stack Overflow Inside ip(6)tables-restore

ip(6)tables-restore存在栈溢出漏洞。是通过fuzz netd守护程序时发现的，它的interface文件是自动生成的。在transaction `Call :: WAKEUPADDINTERFACE`中引用的足够长的字符串编写`in_ifName`，然后调用`wakeupAddInterface`。 最后，它会在`add_param_to_argv`函数中触发堆栈溢出漏洞。

![image-20200706163537880](/images/2020-08-14/fig7.png)