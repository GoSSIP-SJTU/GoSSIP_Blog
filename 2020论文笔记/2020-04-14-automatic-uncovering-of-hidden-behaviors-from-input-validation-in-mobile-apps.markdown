---
layout: post
title: "Automatic Uncovering of Hidden Behaviors From Input Validation in Mobile Apps"
date: 2020-04-14 19:11:48 +0800
comments: true
categories: 
---

> 作者：Qingchuan Zhao∗, Chaoshun Zuo∗, Brendan Dolan-Gavitt†, Giancarlo Pellegrino‡, Zhiqiang Lin∗
> 
> 单位：∗The Ohio State University, †New York University, ‡CISPA Helmholtz Center for Information Security
> 
> 会议：S&P 2020
> 
> 原文：[Automatic Uncovering of Hidden Behaviors From Input Validation in Mobile Apps](https://trouge.net/papers/inputscope_sp2020.pdf)

# 简介

Input validation是指移动应用程序处理和响应用户输入的数据的方式。这篇文章作者验证了Android移动应用可以通过对输入验证行为的分析来揭露其隐藏的功能。

贡献：

1. 两个发现：发现输入验证可以被用来暴露输入触发的secret（e.g. 后门和黑名单secret)；依赖输入的隐藏功能应用广泛。
2. 系统化工具INPUTSCOPE, 可以自动识别执行上下文和输入验证中验证的目标内容，用来揭露输入触发的机密。
3. 在150,000个应用上进行实验。发现8.47%包括后门secret（ secret access keys, master passwords, & secret commands），2.69%包含黑名单secret（禁词）

<!-- more -->

## 背景

### 输入的类型：

1. 内部输入：从程序代码（e.g. 硬编码）中输入，从资源文件（e.g. db）中输入。
2. 外部输入：
   1. 外部本地输入：用户击键， 从系统库输入（e.g. GPS库），从其他本地app使用intent传递的输入。
   2. 外部远程输入：远程服务器或者外部设备（e.g. 蓝牙设备)。

### 如何验证一个输入

用户是否知道允许的输入：

1. 黑名单：对比输入中需要被阻塞的内容。用户不知道完整的名单，并且名单一般不是固定的（可以动态增加），名单一般时秘密的。例子：反病毒签名。
2. 白名单：对比输入中可以允许的内容。白名单是用户知晓的，并且一般是与一个固定的大小绑定的。

---

两者可以同时使用：

1. 句法验证：验证输入的格式和大小，e.g. 无效的电子邮件，电话号码，zip code。
2. 语法验证：验证输入的意义，e.g. 无效时间（2.31），支付金额<0。

### 实例

**后门机密**: app里可以绕过访问控制（e.g. 认证）的输入。

图1是一个由500,000+安装的文件加密应用，用于隐藏或者锁定私人文件，以防被他人访问。第11行的”b***l"就是一个后门机密。

![img](/images/2020-04-14/fig1.png)

图2是一个由1million安装的字典应用，第6行的“q***d"是一个去广告的后门，这个服务本该是付费的。

![img](/images/2020-04-14/fig2.png)

**黑名单机密**：

图3是个在Google Play上由50,000次安装，加上别的商店总共有1.1m次安装的应用。该应用进行了混淆，为了可读性作者把一些方法名进行了替代。第4行v0是用户输入，第15行v2是从是从本地文件中读出的值，与输入对比后有相同的内容就返回true，即validata_nickname返回非法字符的error。所以暴露了intercep_word.txt是app使用的黑名单，可以被提取出来。

![img](/images/2020-04-14/fig3.png)

## 系统设计

INPUTSCOPE工作流程如图4所示。

Input Validation Detection: 使用静态污点分析检测验证的行为

Compared Content Resolution: 从taint sink进行后向切片，识别对比的源和计算最终的string。

Comparison Context Recovery: 输入是用户输入和比较的内容，恢复代码中调度的行为。

Secret Uncovering: 使用具体的规则寻找secret。

![img](/images/2020-04-14/fig4.png)

简化：

​	1. 只关注Java bytecode层的输入验证，不包括native库的。

​	2. 用户输入只关注通过EditText的输入。

​	3. 特别关注使用equals类型的对比。

系统可以处理一些常见的混淆，比如变量和类名的重命名；但是不能处理system API的混淆（e.g. 利用反射），以及使用私有的API进行字符串操作和对比。

### Input Validation Detection

source包括3个系统API。

sinks包括一些直接验证是否相等的API（e.g. equals)，以及间接验证的API（e.g. Map.get）。

![img](/images/2020-04-14/tab1.png)

### Compared Content Resolution:

由于在taint sink对比的字符串不一定直接可见，所以先运行一个后向切片去识别字符串是如何生成的，再使用string value分析获取最终计算得到的值。

**静态后向切片**：首先生成节点是指令，边反向的ICFG (inter-procedural control-ﬂow graph)，然后通过其生成IDFG (interprocedural data-ﬂow graph)。它从目标变量使用点出发，生成点结束。因为一个用于对比的字符串可以来自不同的源并且值可以以不同的方式生成（e.g. 从一个本地文件，或者一个远程服务器的响应)，所以需要以不同的方式解决。

1. 从程序代码中获得字符串的值。后向切片在getString之类的API停止。

  2. 从资源文件中获取字符串的值。三种资源文件：files(e.g. text files, JSON files)，数据库(e.g. SQLite databases)，key-value stores（e.g. sharedPreferences)。首先处理文件名和文件相关的语义，再获取值。比如SharedPreferences对象，需要先获取文件名，然后从“key”得到string的生成。

  3. 如果是外部输入，后向切片不会处理具体的值，因为外部输入的值只能真实运行。

     ![img](/images/2020-04-14/tab2.png)

进行后向切片时会维护一个IDDG （inter-procedural data-dependency graph），用于记录数据流路径上相关字符串的计算序列，这个序列会在最后重构最终字符串的时候使用。

**字符串值的分析**：这一步使用之前作者开发的开源工具LeakScope，在不进行真实执行的情况下仿真字符串的相关计算。作者通过IDDG中原始的执行序列进行前向的字符串值的计算，仿真所有的字符串操作的API。比如说，如果字符串操作中有substring，作者就会生成这个子字符串的值。

> Why does your data leak? uncovering the data leakage in cloud from mobile apps

**裁剪用户已知的比较值**：本文的目标时揭露隐藏的行为，如果对比的内容是从可视的用户接口中输入的，或者用户输入前就被系统API动态加载 (e.g. EditText.setHint)，就裁剪掉。

### Comparison Context Recovery

用户输入的代码调度有两个属性：1）在一个方法的判定块里一个用户输入被验证了多少次；2）有多少个潜在的分支，根据这两个属性分为了以下三种：

1. One-to-Two调度。例如图2中的if。
2. Many-to-Two调度。例如图3中的调度，一个用户输入是和一个数组里的每个元素对比，但是只有两个分支。
3. Many-to-Many调度。依赖不同的对比结果有不同的分支，switch-case一般就是这种调度。

### Secret Uncovering

作者从以上3中调度中定义了4种规则去揭露四种类型的隐藏行为：secret access key, master access password, secret command, blacklist secret。

1. 从one-to-two调度中揭露隐藏行为。

这种情况下一般是使用用户输入去解锁一个行为，这样的用户输入可以看作是secret access key。

这种行为可能也是一些普通的服务，比如在解密游戏中用户需要输入一个正确的答案进入下一关。但是作者认为解密游戏的结果一般会是可变的 (e.g. 从服务器返回)，而不是硬编码在程序中（导致作弊）。

**识别secret access key**：(1) one-to-two调度中的用户输入 （2）比较值是硬编码在app中

2. many-to-two中的秘密

在这种情况下，满足不同验证的输入可能会导致同一个结果，并且对比的内容可以从不同的源获取，也可以从相同的源获取。

​		1）从不同的源获取：如果比较的内容来自多个来源，则此类型的比较说明了一种情况，在该方法中，如果用户输入等于多个来源中的任何值，则程序将执行相同的操作。 换句话说，每个比较值都可以覆盖其他值。如果比较值其中之一是硬编码的，它就可以被用来覆盖其他的值使得程序到达相同的状态。图一就是一个例子，这个隐藏的硬编码字符串就被认为是一种master key。

**识别master password**: (1) many-to-two调度；（2）比较值从不同源获取；(3) 比较值之一是硬编码。

​		2) 从相同源获取：这种情况一般是程序拥有一个list，需要对list中每一个项目都进行对比是否等价，每个等价结果会导致相同的程序行为。这种被定义为blacklist。

**识别blacklist secret**: (1) many-to-two调度；(2) 比较值从同一个源取得。

3. many-to-many中的秘密

这种调度中不同的比较值会导致不同的结果。这种行为就像是一个终端接收不同的命令去执行不同的操作，所以定义为secret command。

**识别secret command**: (1) many-to-many调度；（2）比较值包括多个硬编码的字符串。

## Evaluation

使用INPUTSCOPE进行大规模分析的结果如图3所示。包括top 100,000个Google Play上的应用，top 20,000个百度商店的应用， 30,000个三星的固件上提取的预装应用。

![img](/images/2020-04-14/tab3.png)

1. **隐藏的后门行为**。

   ​	（1）**被secret access key触发**。作者从安装量超过1百万的应用中随机选择了30个app进行手工分析，并总结出3中常见的使用类型，如图5中列出了每种类型安装量最高的5个应用。

![img](/images/2020-04-14/tab4.png)

登录管理员界面。一个安装量超过5百万运动类应用成功登录后，管理员界面允许攻击者执行特权操作，例如更改配置URL，更改网络ID或重置“临时密码”。

重置用户密码。一个安装量超过5百万的锁屏应用，攻击者可以多次输入错误的密码触发一个隐藏的按键，通过点击这个按键输入一个特殊的密码后就可以重置密码。

绕过高级服务的付款。如例子中去掉广告。

​		（2）**被master key触发**。作者随机选择了10个进行分析，结果如表5，其中一个是帮助用户在手机丢失是锁定内容，一个是加密日记，都可以通过master pwd绕过。

![img](/images/2020-04-14/tab5.png)

​		(3) **被secret command触发**。选择了结果中安装量前10的应用分析。这种一般是用于debug或者其他的功能（擦除缓存数据等）。

![img](/images/2020-04-14/tab6.png)

 2. **隐藏的blacklist secret**。作者分析了20个应用。

    ​	宏观结果：

    ![img](/images/2020-04-14/tab7.png)

    微观结果：

![img](/images/2020-04-14/tab9.png)

## Discussion

### 准确性

作者手动检查了70个app，其中发现了1个误分类和8个误报，正确率87.14%。误分类是把secret command分类成了blacklist secret，误报中有2个是良性的隐藏行为。

### 局限性和未来工作

1. 对webview组件的支持。
2. 无法识别自定义字符串操作。
3. 对于从数据库中提取的数据，由于需要数据库的查询和数据库的结构语义，无法完全进行处理。(Static detection of second-order vulnerabilities in web applications)。
4. 无法区分良性的隐藏行为。

