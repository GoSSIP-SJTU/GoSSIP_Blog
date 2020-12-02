---
layout: post
title: "Total Recall: Persistence of Passwords in Android"
date: 2019-04-16 13:38:51 +0800
comments: true
categories: 
---

出处：NDSS 2019  

作者：Jaeho Lee, Ang Chen and Dan S. Wallach

单位：Rice University

原文链接：https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_06A-4_Lee_paper.pdf

<hr/>

#### 文章概述
处理内存中敏感数据，比较安全的做法是一旦数据不再使用，便将敏感数据缓存清零。这样即使攻击者获得了设备物理内存快照(通过物理接触或者Meltdown、Spectre等攻击方式)，也没办法获取到敏感数据内容。在本文中，作者将关注点放在内存中的用户密码(password)上，通过对安卓框架和不同App(包括流行App、密码管理App和系统锁屏程序)的深入分析，对安卓App的内存中密码数据残留情况进行了系统的研究。研究结果表明安卓App中密码数据残留的情况确实不容乐观。作者总结了各种密码数据残留的情况及其成因，提出了解决方案并对该方案进行了评估。

<!--more-->

#### 研究背景
##### 1. Authentication in Android
+ Remote authentication: C/S架构
+ Local authentication: 主要指对本地敏感数据读取的认证，涉及的软件包括1Password、LastPass等密码管理软件
![](/images/2019-04-16/1.png)

##### 2. 可能导致残留的原因
+ 后台应用: App进入后台后，GUI被隐藏，但其进程、内存仍然保留。
+ 缺乏对密码的特别保护：尽管安卓平台引入了TrustZone和配套的Keystore来保护敏感数据，但很少有App真正使用了TrustZone来管理其密码。
+ 延迟的垃圾回收：Java Gabage Collector(GC)
+ Native Code：如果密码从Java层传到了Native层，可能残留在Native层中。
+ Long chain of custody：用户的输入可能在多个进程间有残留，如输入设备驱动->键盘App->目标App


#### 验证密码残留的广泛存在
+ 威胁模型：攻击者可以获取、分析设备的内存快照。
+ 实验方法：挑选11个App，安装、启动App(分别在Android 6、7、8三个版本的系统上)，输入密码进行认证，然后将内存dump出来，共dump四次：
    + 刚刚进行认证后
    + App切换到后台后
    + 从YouTube中播放视频后
    + 将设备锁屏后
+ 结果(4 Observations)
    + 所有的App都存在密码残留问题
    + Android 6、7、8版本上都存在密码残留问题
    + 已有一些开发者关注了密码残留问题：一些App进入后台后密码被抹除
    + 密码字符串在内存中很容易识别
    ![](/images/2019-04-16/8.png)
![](/images/2019-04-16/2.png)



#### 分析密码残留的根本原因
##### 安卓框架
两个关键技术：Runtime logging和Password mutation
![](/images/2019-04-16/3.png)
+ Problem #1：缺少安全的清零措施
    与输入密码密切相关的TextView在App的生命周期中不提供内存清零的功能，因而这部分工作依赖于开发者。
+ Problem #2：不安全的SpannableStringBuilder类
    TextView中的缓存都是SpannableStringBuilder类的对象，该类的实现会导致两个问题：
    + 每当用户输入一个字符，SpannableStringBuilder都会分配一个新的数组，并将已输入的内容拷贝进去，丢弃原来的数组(但不清零)
    + SpannableStringBuilder提供clear方法，但该方法仅仅将数据赋值为null，而不是将数据清零。对开发者也存在误导。
+ Problem #3：缺少安全的getPassword() API
    > "Objects of type String are immutable, so there is no way to overwrite the contents of a String after usage"

    TextView中为了获取用户输入，提供了getText()这一方法，返回CharSequence对象。该对象是String类的接口(interface)，开发者通常将其作为一种String类，并调用getText().toString()方法获取String类型的密码。
    相比之下，Java Swing UI library中提供getPassword()方法替代getText()方法，返回字符数组。但安卓中并没有相应的getPassword() API。

##### 键盘(输入法)App
+ Problem #4：缓存最近的输入
    LatinIME(AOSP默认键盘)将最近的输入缓存，直到有后续的输入才将缓存清除。例如，在先后输入用户名和密码时，先缓存用户名，在开始输入密码时，将用户名的缓存清除。但在用户输入密码后，密码的缓存便一直停留在内存中。
![](/images/2019-04-16/4.PNG)


###### 第三方Apps

![](/images/2019-04-16/5.png)
+ Problem #5：使用、传播String类型的Password(Sample 1a)
+ Problem #6：缺少手动的TextView清除(Sample 1b)
+ Problem #7：缺少App层面的清零

#### 解决方案
+ SecureTextView：修复安卓框架
    + 修改TextView，对password数据进行特殊处理
    + 实现安全版本的SpannableStringBuilder
    + 监听手机、App状态的改变
+ KeyExporter：修复安卓App
    开发者需要遵守的规范：
    + 使用字符数组来获取TextView中的密码数据(Use charAt() instead of toString())
    + 使用TextView的clear()函数清空缓存
    + 在较早阶段使用密码生成强密钥(减少密码的传播)
    + 及时将密码相关的内存清零

    但开发者的安全意识、技能参差不齐，所以有必要为其提供帮助。作者通过修改TextView()，添加getKeyExporter() API，提供Key derivation功能减少密码的传播，并处理好内存清零的操作。
![](/images/2019-04-16/6.png)

#### 方案评估
+ 作者提出的方案很有效，可以移除测试App中所有的密码残留
+ 对App的修改通常来说比较小
![](/images/2019-04-16/7.png)

#### 总结
文章的思路很清晰，提出问题->验证问题的存在->发现其原因->提出解决方案->对方案进行评估，但对于Runtime logging和Password mutation这两个技术细节的实现方式没有讲得很清楚。