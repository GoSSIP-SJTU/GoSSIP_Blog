---
layout: post
title: "Fear and Logging in the Internet of Things"
date: 2018-07-11 18:53:41 +0800
comments: true
categories: 
---

作者: Qi Wang, Wajih Ul Hassan, Adam Bates, Carl Gunter

出处: NDSS'18

单位: University of Illinois at Urbana-Champaign

原文: http://seclab.illinois.edu/wp-content/uploads/2017/12/wang2018fear.pdf


作者观察到IoT平台存在一种新的威胁，当多个app和多个智能设备绑定在一起时，我们很难定位某个设备被攻击或者配置错误的原因。那首先想到的方法就是查看日志，然而存在的日志只记录了设备在某事某刻发生了什么，但不能解释不同事件之间的因果关系，比如是谁通过何种途径触发了设备状态的改变。因此，本文以Samsung的SmartThings为例，使用数据溯源的技术，来追踪IoT事件发生的一连串因果关系。

<!-- more -->

## 背景
### IoT Platforms and Smart Home Platforms
截至2017年，大概有450个IoT平台

这些平台支持
* enable custom IoT applications
* create a device abstraction
    * Device Handler (SmartThings)
    * Device Shadows (AWS IoT)

Smart Home可分为

* cloud-centric (更流行)
* hub-centric 
   
![](/images/2018-07-11/15227311988625/15227631065582.jpg)

新的安全威胁

* 多个app与多个device对应
    * 访问敏感数据
    * 越权操作
* 如何检测非预期事件产生的原因是个困难问题

#### Samsung SmartThings
![](/images/2018-07-11/15227311988625/15227670280560.jpg)

* the SmartThings cloud backend
    *  SmartApps
        *  允许开发者创建自定义的automation
    *  Device Handlers
        *  硬件抽象，虚拟设备
* the hub
    * 与device交互并且转发云和device之间的通信
* the SmartThings mobile app
    * 接收设备状态，远程控制设备

SmartApp
![](/images/2018-07-11/15227311988625/15227673327321.jpg)

Device Handler
![](/images/2018-07-11/15227311988625/15228073580661.jpg)


### Data Provenance
数据溯源记录了一个数据对象从创建到现在的所有的行为，可以用来解释“数据是在什么环境下产生的”和“这个消息是来自于敏感数据“等问题。数据溯源可以让我们理清智能家居各组件之间的因果关系。
![](/images/2018-07-11/15227311988625/15227635201023.jpg)

#### System Model
W3C PROV-DM

* an **Entity** is a data object
* an **Activity** is a process
* an **Agent** is something bears responsibility for activities and entities

* Dependency
    * which entity *WasAttributedTo* which agent
    * which activity was *WasAssociatedWith* which agent   
    * which entity *WasGeneratedBy* which activity
    * which activity *used* which entity
    * which activity *WasInformedBy* which other
activity
    * which entity *WasDerivedFrom* which other entity


 

## 威胁模型
* API-level attacker可以访问并修改device的状态

这些能力来自于

* Malicious Apps
* Device Vulnerability
* Proximity

假设

* device is integrity
* cloud is trusty

## ProvThings
需求

* Completeness
    * 事件的因果关系、数据的状态  
    * 系统应该能够回答“睡眠传感器是如何产生的数据”等问题
* Applicability
    * 通用   
    * 兼容各种物联网平台
* Minimality
    * 对原有系统的改动尽量小
    * 需要很少或不需要修改平台本身的语义
    
### IoT Provenance Model
W3C PROV-DM模型

举个例子：智能设备产生设备消息(entities)并执行一个命令(activities), 使用DEVICE来区别AGENT的类型
![](/images/2018-07-11/15227311988625/15228169352257.jpg)

有了这个模型，就可以定义source和sink

* source是安全敏感的data object，比如门锁的状态
* sink是安全敏感的method，比如开锁的命令

### Provenance Management Framework
![](/images/2018-07-11/15227311988625/15228180601544.jpg)

模块设计

* Provenance Collectors 收集 provenance records
* Provenance Collectors 合并 records 到 IoT provenance model
* 构建provenance graphs 并且 存储到 database
* policy monitor 分析 graphs 并采取行动
* frontends 用来提供交互的 interfaces 

#### Provenance Collectors
* 根据不同的组件设计不同的provenance collectors
* 利用程序插桩追踪data flow 和 method invocations 
* 这部分是平台相关的，可以使用不同的语言兼容不同的平台

#### Provenance Recorder
* 把provenance records合并、过滤并导入IoT provenance model
* 构建并存储provenance graphs

#### Policy Monitor
* monitor 的输入是根据不同组件创建的 policies
* 验证某个action是否和policy匹配

#### ProvThings Frontends
* 为用户提供一个接口，定义query API
* 创建配置文件，policies规则

配置文件可以是用户是否允许产生provenance records，以及如何收集和处理并存着这些records

### Instrumentation-based Provenance Collection

插桩用来追踪data assignment和method invocation

* 根据源码生成Abstract Syntax Tree (AST) 和 call graph
* 根据AST进行control flow analysis 和 data flow analysis
* 插桩，当插桩代码执行时把record发给ProvThings

为了识别哪些source和sink是安全敏感的

* 对与每个method，首先检查他是不是sink，如果是就看entry point是否可到达这个sink
* 如果这个method是entry point，就对这个method插桩
* 对于每个method的分支，迭代检查是否可达到sink
* 如果到达sink，则对sink的执行进行插桩
* 如果sink使用的变量来自source，计算这个这个变量的sink的后向切片

![](/images/2018-07-11/15227311988625/15228198941663.jpg)

静态的插桩还支持检查运行时的逻辑

* method execution代表activity, callee和caller的关系是WasInformedBy
* method invocation和他的参数是Used的关系，返回值的关系是WasGeneratedBy
* entities之间的数据依赖表示为WasDerived
* 没有直接参与复制的数据标记为Implicit-Used

### IoT Provenance Policy Specification
![](/images/2018-07-11/15227311988625/15228245052963.jpg)

* pattern定义了目标行为的模式
* check定义了检查pattern是否存在
* action定义了当check满足时应该采取什么action

![](/images/2018-07-11/15227311988625/15228257184052.jpg)

当Front Door Lock触发setCode命令时提醒用户

* 在sink被执行之前，插桩的代码向Policy Monitor发出请求， Policy Monitor返回是否允许sink的执行

### Comparison to Other Information Flow Solutions

![](/images/2018-07-11/15227311988625/15228145455228.jpg)


## 实现
![](/images/2018-07-11/15227311988625/15228104895494.jpg)

* 在SmartApps和Device Handlers被提交到SmartThings后端之前，在他们的代码中插桩。

* 插桩代码收集数据记录，并把他们发给ProvThings的服务器，用来审计日志。

### SmartApp Provenance Collector
* 静态源码插桩
* 运行时收集provenance

#### Static Source Code Instrumentation

* 使用Groovy AST transformation生成SmartApp的抽象语法树

* 通过SmartThings的开发者手册人工识别SmartApps的入口，source和sink

    * entry points
        * lifecycle methods
        * event handler methods
        * web service endpoints
    * sources
        * device states
        * device events
        * inputs
    * sinks
        * device control commands
        * SmartThings provided API

特别的

*  dynamic method invocation
*  global variables

![](/images/2018-07-11/15227311988625/15228114565288.jpg)


#### Runtime Provenance Collection
* entryMethod
    * 溯源entry point的调用
* trackVarAssign
    * 变量赋值
* trackSink
    * sink调用
* special type
    * 动态方法调用
    * 状态

### Device Handler Provenance Collector
* entry points
    * lifecycle methods
    * device command methods
    * the parse method
    * web service endpoints

## 评估
* ProvFull (PF), 对所有的指令进行插桩
* ProvSave (PS), 用Selective Code Instrumentation算法有选择的插桩

### Effectiveness 
从调查中总结出26种可能的攻击。

* 12个基于漏洞报告 
* 14个基于病毒分析

### Instrumentation overhead
* 236 SmartApps, 平均280行代码
* 132 Device Handlers, 平均200行代码

![](/images/2018-07-11/15227311988625/15227570730652.jpg)

### Runtime Performance
* ProvSave 20.6% overhead
* ProvFull 40.4% overhead
![](/images/2018-07-11/15227311988625/15228266613160.jpg)

#### Storage Overhead
168,000 events

* ProvFull 产生 219 MB
* ProvSave 产生 89 MB
![](/images/2018-07-11/15227311988625/15228266883375.jpg)

### Query performance
For graphs with 2 million nodes

* ProvSave, 请求所有节点的平均时间是2 ms，请求所有sink activity的节点的平均时间是9 ms

* ProvFull, 请求所有节点的平均时间是5 ms，请求所有sink activity的节点的平均时间是14 ms

![](/images/2018-07-11/15227311988625/15228267004337.jpg)


## 使用场景
### Professionals
#### Platform developers
Alice

* smart home customer

WhenEveryoneIsAway

* 当家里没人时，设置mode为Away

LockItWhenLeave（malicious）

* 订阅该mode的状态，当mode为Away时锁门并打开摄像头

当Alice回家时发现门是开着，要求Admin调查原因。。

![](/images/2018-07-11/15227311988625/15227541861708.jpg)
当一个sink被动态方式调用，并且sink的值来自HTTP，会向Admin报告这个隐患。
![](/images/2018-07-11/15227311988625/15227542008861.jpg)

#### Help Desk Staff

当一个顾客使用SmartThings打开厨房的灯，但是灯有时会突然关掉。她想知道是硬件问题还是软件的问题。

她会去检查日志，如果有off的记录，那么就是SmartApp的问题。否则就是硬件问题。


### Techies
Bob

* smart home user

LockManager

* 允许用户更新或删除pin code

FaceDoor（malicious）

* 允许用户通过面部识别解锁  

假如SmartThings存在越权漏洞， FaceDoor可以订阅所有的事件，那么它就可以偷取敏感信息，比如LockManager的pin code

![](/images/2018-07-11/15227311988625/15227521830546.jpg)


### Typical Consumers
SmokeMonitor 

* 正常的程序
* 监控smoke detector的状态
    * 打开fire sprinkler
    * 打开窗户
    * 触发报警器 
    
  
SmartLight

* 插入恶意payload
* raise a fake physical device event for smoke detector
    * physical damage
    * 窃贼从窗户进来。。

生成provenace图，这里对与smoke handler来说，并不能区别event来源于smartLight还是smoke detector  
![](/images/2018-07-11/15227311988625/15227449028779.jpg)

这张图user可能看不懂，所以简化一下。。

WhyThis

* 让用户决定是否允许未知行为的发生
* open is a sink function   
* 查看行为的解释
![](/images/2018-07-11/15227311988625/15227457219788.jpg)

如果用户选择了deny，之后会默认使用这条规则拒绝所有类似event
![](/images/2018-07-11/15227311988625/15227459401658.jpg)


### Privacy Considerations
* 为了保护消费者的隐私，消费者可以选择provenance收集的粒度以及这些数据可以分享给谁。
* 平台通过访问控制管理访问provenance的权限。

## End

