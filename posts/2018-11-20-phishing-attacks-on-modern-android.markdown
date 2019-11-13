---
layout: post
title: "Phishing Attacks on Modern Android"
date: 2018-11-20 16:38:47 +0800
comments: true
categories: 
---

作者：Simone Aonzo, Alessio Merlo, Giulio Tavella, Yanick Fratantonio  

单位：University of Genoa, EURECOM

出处：CCS 2018 

原文：http://www.s3.eurecom.fr/~yanick/publications/2018_ccs_phishing.pdf

----
#### 文章概述
安卓系统为了便利性引入了许多新的功能。这篇文章中，作者对现有的四款比较著名的密码管理App以及安卓系统提供的Google Smart Lock(GSL)服务进行了分析，并利用密码管理器(Password Managers， PMs in short)和即时App(Instant App)这两个新功能在设计上的不足针对自动填充密码功能实现了安卓系统上的全新钓鱼攻击。该种攻击方式比现有的钓鱼攻击方式更容易实现。在此基础上，作者设计了新的API来避免该种钓鱼攻击方式的发生，但他也表示自动填充密码功能的安全实现需要整个开发者社区共同努力。
<!--more-->

#### 研究背景
##### 1. 移动端密码管理 (Mobile password managers)
密码管理技术最早在Web端被广泛使用，可以帮助用户针对不同的网站创建不同的伪随机密码，并在用户登录时根据域名提示并填充相应的登录密码，在提供便利性的同时减少简单密码、易猜测密码、跨站使用相同密码等不安全现象的发生。随着移动平台的发展，密码管理技术也被移植到移动端，并以App的形式提供类似的管理、提示并自动填充密码的功能。如果要实现自动填充功能，必然涉及到App之间的交互，然而安卓的沙盒机制对此有所限制。目前来看有三种机制能够帮助绕过沙盒的限制，实现这一功能：
1. Accessibility Service(a11y, in short)
  a11y本意是用来帮助残疾人士的，它使得运行在后台的App能够在“Accessibility Events”出现的情况下接收来自系统的回调，同时App也可以获取一些目前运行的App的一些上下文信息，包括App的包名等。开发人员利用这一功能开发出很多新的用途，其中提示并自动填充密码就是其中之一。PMs利用a11y来确定用户在使用哪一App、当前界面是否有可以填充凭证(credentials)的字段，并利用它来和目标App进行交互。然而，近些年很多研究表明a11y也被用来进行窃取用户个人信息等恶意操作。因此，Google也在系统层面上开发出新的Autofill Framework帮助开发者实现自动填充密码功能。
2. Autofill Framework
  Autofill Framework在安卓8.0之后被引入，它可以让PMs确定用户正在和哪一个App进行交互，也可以让App利用编程方式填充凭据字段。但这要求想要适配的App申请BIND_AUTOFILL_SERVICE权限并补充相关的XML属性。
3. OpenYOLO(YOLO stands for "You Only Login Once")
  OpenYOLO协议是Google和Dashlane共同开发的协议，协议包含客户端和服务端两部分，不依赖于a11y和Autofill Framework,但它需要对那些支持提示并自动填充密码功能的App进行改写，在已有代码中增加OpenYOLO客户端。实际运行时，客户端通过Intent等机制和服务端(在适配OpenYOLO协议的PMs中)进行通信获取相关凭证，具体如下图所示。
  ![](/images/2018-11-20/1.png)

然而，根据作者的研究，这三种机制在设计上存在一个通病：利用App的包名作为识别App的主要信息。这也是之后作者设计的钓鱼攻击能够成功的原因之一。

##### 2. 即时App (Instant App)
即时App技术由Google实现，允许用户在不安装App的情况下进行“试用”。开发者需要上传他们App的一小部分,也就是Instant App, 并附上一个URL pattern，谷歌会利用Digital Asset Links协议验证URL pattern所属域名和App是否互相对应。实际应用场景中，用户点击某一URL并确认后，Instant App(小于4M)即被下载并运行在用户的设备上，功能上类似国内的微信小程序，但在实现技术上有所不同。
然而这种技术能够让攻击者在不需要安装App的情况下对用户设备UI拥有完全的控制，更便于恶意App伪装为真正的App,进而使得作者设计的钓鱼攻击在实际场景中更容易实现。

#### 案例分析
##### 1. The Mapping Problem
和Web端可以直接利用域名来确认服务端身份不同，在移动端PMs需要建立起App与其关联的网站之间的映射(Mapping)才能确定应该提示并填充哪一个凭证。然而PMs所建立映射存在的问题可能导致钓鱼攻击的成功，因此作者对4款著名PM App： Keeper、Dashlane、LastPass、1Password以及GSL进行了人工分析，指出了各自存在的问题。这也为钓鱼攻击的成功实施做好了准备。

##### 2. 分析步骤
1. 确定是否使用目标App的包名作为判断是否提供凭证填充的唯一信息
2. 如果确定包名是唯一信息，则进行逆向分析明确包名和域名之间的映射规则
3. 尝试利用映射规则存在的漏洞进行攻击

##### 3. 分析详情

+ Keeper
  Keeper有超过一千万的下载量，支持a11y和Autofill Framework。在获取到目标App的包名后，Keeper尝试在Google play Store中访问对应包名的网页(比如对于包名com.facebook.katana, 对应的网页为 https://play.google.com/store/apps/details?gl=us&id=com.facebook.katana)，如果网页存在，Keeper对返回的网页进行解析并找到其中“开发者网站(app developer website field)”字段，并将其作为该包名对应的域名。然而这种映射方式很容易被攻击者利用，因为“开发者网站”字段的信息并不能被信任，App发布者上传了自己的App后可以任意修改该字段。
+ Dashlane
  Dashlane有超过100万的下载量，支持a11y、Autofill Framework和OpenYOLO。它硬编码了包名-域名的映射，其中包含81个条目。初次之外，对于不在其中的包名，它将包名按点分开为三个部分(aaa.bbb.ccc分为aaa,bbb,ccc),对于每一部分，检查是否存在至少三个字符包含在它硬编码映射的网站字段中，比如xxx.face.yyy或xxx.facts.yyy和facebook.com对应。这种方式很容易被攻击。
+ LastPass
  LastPass也有超过100万的下载量，支持a11y、Autofill framework和OpenYOLO。对于包名为aaa.bbb.ccc的App，它将其和以bbb.aaa结尾的域名映射起来。这种方式很容易进行攻击，因为攻击者可以任意设置恶意App的包名，只要不和应用市场中的其他包名重复即可。此外LastPass也提供了一种"crowdsourced mapping"的映射方式，但对这种方式只存在理论上的攻击可能。
+ 1Password
  1Password也有超过100万的下载量，支持a11y、Autofill framework和OpenYOLO。它不进行映射，而是将整个凭据列表展示给用户由其自行选择。在丧失了便利性的同时也增加了安全性，不易于攻击。
+ Google Smart Lock
  GSL在Chrome上集成了PM功能，因此也被作者作为分析的对象。GSL的映射相对安全，因为需要开发者提供很多信息来保证映射的正确。但该种方式相对麻烦，如果Google能够公开它的映射数据库，那会非常有利于PMs安全性的提升。

#### 实际攻击
结合密码管理和即时App两个功能的缺陷，作者进行了如下的攻击。
假设如下图(a)中场景，用户访问网站，并有仿冒的Facebook的“Like”按钮，按钮链接到攻击者掌控的Instant App。一旦用户点击了Like按钮，会有图(b)中请求用户确认启动Instant App的提示，该Instant App使用“Open With”作为名称，图标为纯白色方框，这就误导用户点击“OPEN APP”按钮，并自动下载Instant App。经过图(c)所示等待页面(约一秒)后，如图(d)，App启动。因为app包名被设置为com.facebook.*形式，LastPass自动提示使用真正的Facebook凭证，一旦用户点击该悬浮框进行确认，那么凭证便会泄露到攻击者手中。
在此基础之上，作者也评估了这些PMs对于隐藏的密码输入框是否有抵御能力。作者用如下几种方式将密码输入框“隐藏”并成功获取凭证：

+ 透明度alpha设置为0.01，设置为0则会失败
+ 大小设置为1dp * 1dp
+ 文字颜色与背景色相同(只对a11y有效)
+ 设置invisible标签(只对Autofill Framework有用)
  ![](/images/2018-11-20/2.png)
#### 保护方式
能够抵御这种攻击的方式有很多，作者提出getVerifiedDomainNames()这一API，能够帮助PMs更好地确认App所对应的网站。该API具体实现逻辑如下图所示。但API要想正常工作还需要整个开发者社区的共同努力。
![](/images/2018-11-20/3.png)
#### 总结
文章提出一个全新的钓鱼攻击方式，从新的功能出发，挖掘其中的问题。这也提示我们要关注Android系统不断更新的一些新功能，新功能在提升便利性的同时也可能带来新的安全问题。但是相对于一些工作分析了超过20个PM App文章只分析了4个App，有些结论的一般性还需要进一步论证。文章一些钓鱼攻击的技术，如伪造App名称等也是值得学习的。
