---
layout: post
title: "The Rise of the Citizen Developer: Assessing the Security Impact of Online App Generators"
date: 2019-01-17 19:27:39 +0800
comments: true
categories: 
---

出处：S&P 2018  

作者：Marten Oltrogge, Erik Derr, Christian Stranksy, Michael Backes等

单位：CISPA等

原文链接：https://saschafahl.de/papers/appgens2018.pdf

----
#### 文章概述
越来越多的App都由在线应用生成器(online application generators, OAGs for short)自动生成、分发、维护，这降低了对开发人员的技术要求，因而吸引了不少业余开发者(citizen developers)。然而，使用这类工具可能导致的安全问题尚未被研究。如果使用这一类工具生成的App存在安全问题，那么将会对整个移动应用生态系统的安全性造成影响。在本文中，作者对OAGs进行了系统、深入的分析，主要工作包括：
+ 通过对常用OAGs所支持的工作流、平台、App生命周期等特征进行分析，(首次)对常用OAGs进行分类。
+ 分析不同OAGs所生成App的指纹，从而确定OAGs的市场占有情况。
+ 综合利用动态、静态、人工分析，研究OAGs的安全性。

<!--more-->

#### OAGs简介

##### 1. App生成器(AppGens)
App生成器是将App开发、分发、维护等阶段部分、甚至完全自动化的工具，它的优势主要有四点：
+ 替开发者抽象出实现的细节，而只关注App的行为、功能
+ 除了核心的App生成，还提供App签名、分发等功能
+ 生成的App可以支持多平台
+ 将用户管理、用户登陆等功能模块化

App生成器主要有三种不同的工作方式：
+ 单独的框架(Apache Cordova、PhoneGap、Xamarin)
    这一类工具实现了App可能使用到的核心功能，并为开发者提供接口，方便进一步地定制。框架的输入是与平台无关的语言(如JavaScript、HTML、C#等)所写的代码，并将其打包成App。此类工具也提供丰富的插件，基本覆盖常见的功能需求，但对于生成App之后的App分发等环节没有提供帮助。

+ 在线服务(Online Service or OAGs, 如Andromo、Biznessapps)
    这一类工具无需写任何定制代码，使用者所需要做的便是在Web端利用所见即所得的方式添加UI元素，不同的UI元素便对应App内不同的组件(如登录表单、内置浏览器、二维码扫描器等)。对于用户管理、用户登录、传输数据到后端等常见操作，提供整套框架。因而没有App开发经验的人也可以使用此类生成器定制自己的App。同时，在线服务也提供分发App、用户数据分析等功能。

+ Developer-as-a-Service(CrowdCompass、QuickMobile)
    开发者不需要处理任何技术部分的问题，只需要提出自己的需求。剩下的只需要由生成器团队负责实现(类似外包)。此类服务往往都是针对某一类App，如会议App、活动App等。

##### 2. OAGs分类
分类之前，需要明确有哪些主流的生成器。作者利用Goolge检索了"app maker android"、"android generate app"、"{business, free, diy, mobile}(android) app {generator, creator, maker}"等词条，选择前五个搜索结果并去重。此外，作者还在Appindex、Quora、Werockyourweb、Businessnewsdaily等网站检索生成器相关的内容。最终的生成器列表如下表所示(去除了无法明确其生成App特征的生成器)，主要从以下四个维度进行分类：
+ Freeware：是否免费
+ Multi-platform support：是否提供跨平台支持
+ Components：是否支持任意添加组件
+ Publishing support：是否支持App的签名、分发

![](/images/2019-01-17/1.png)
##### 3. 市场占有率分析
为了量化OAGs在整个App应用市场的占有率，作者下载了Google Play上2291898个App。手动逆向了使用生成器生成的App和一个baseline app(不包含任何三方库、仅有一个“Hello World” activity)，利用差分分析的方式，提取了可以用来识别不同OAGs的App特征：
+ App包名
    Tobit: com.Tobit.* 
    Andromo: {com|new}.andromo.dev*
+ 代码命名空间(Code Namespaces)
    Andromo: com.andromo 
    Tobit: com.Tobit.android.slitte.Slitte
+ App签名、签名主体
    AppYet: /C=CA /ST=ON /L=Oakville /O=AppYet /CN=www.appyet.com
+ 文件目录、内容
    AppyPie: 在assets/www目录下包含appypie.xml文件
    AppsGeyser: 在res/raw/configuration.xml文件中包含URL"appgeyser.com"

利用上述特征，作者可以对2291898个App进行分析，并最终发现其中至少有255216(11.14%)个是由OAGs生成的，这些App总下载量超过11.4亿。此外，5款最流行OAGs占所有OAGs占有率的73%。下表所示为其具体数据。
![](/images/2019-01-17/2.png)
#### OAGs(Online Service)安全性分析
##### 1. 安全分析方法
基于OAGs的工作原理，作者假定：使用同一个OAG生成的App拥有相同的样板代码，因而这些App存在的漏洞也相同。所以需要先了解样板代码的生成逻辑。作者选择了13个最流行的OAGs，使用了它们的服务，分别创立了如下三个App：
+ App1: "Hello World" App，最小App
+ App2: 在最小App的基础上增加HTTP、HTTPS请求功能
+ App3: 实现用户登录或向服务器传输数据功能

随后，作者分析了这些生成的App，发现样板代码主要有两种模式：
+ 完全样板代码
    App中包含所有模块的代码、逻辑等，App之间的差异体现在配置文件上，配置文件包含了App所需的数据，决定了App的功能。
+ 分模块的样板代码
    Andromo和Appinventor只在App中添加开发者需要的模块。

##### 2. 针对OAGs的攻击
综合使用静态分析、动态分析的方式，作者发现了两类攻击方式：
+ 重配置攻击(Application Reconfiguration Attacks)
    在十一个使用完全样板代码的OAGs中有5个是仅支持静态或仅支持动态加载配置文件，另外六个同时支持两种方式。对于静态配置文件，作者发现其中七个都是明文存储，可以直接读写。有一个虽然加密了，但密钥硬编码在代码中，可以直接破解。而对于动态加载的配置文件，作者发现只有三个使用了SSL，其余都使用HTTP协议传输配置文件，很容易受到中间人攻击。

+ 基础设施攻击(Application Infrastructure Attacks)
    许多OAGs都将它们的Web服务和生成的App绑定起来。因此，作者利用Qualys SSL Labs(在线分析网站)去分析这些OAGs的Web服务是否能够防止SSL/TLS攻击。结果显示，这些OAGs存在使用HTTP协议、使用自签名证书、证书过期等问题。此外，OAGs将自己的API密钥硬编码在代码中，这也属于不安全的做法。
![](/images/2019-01-17/4.png)
##### 3. 其他安全问题
除了上述两个问题，作者还研究了以下三个问题：
+ 权限设置问题
    App最好只申请需要的权限，同时也不应当暴露不必要的组件。基于此，作者利用PScout、Axplorer等工具，结合手工分析，发现所有OAGs生成的App都申请了过多的权限，同时除了Andromo、AppInventor、Biznessapps三个OAGs，其余OAGs都存在暴露组件的情况，如Seattle Cloud生成的App暴露了InternalFileContentProvider组件，可能导致攻击者能够读取App使用过的文件，包括shared preferences文件、数据库等。
+ 密码学误用
    作者定义了密码学误用的规则，包括过时的算法、对称加密使用ECB模式、硬编码密钥、不安全的哈希算法等，检测结果表明7个OAGs存在密码学误用的情况。
+ WebViews安全问题
    针对WebViews的安全，作者制定了以下几个规则：
    + 分别追踪HTTPUrlConnection和HTTPSUrlConnection的参数，使用HTTP则判为不安全，使用HTTPS则判为安全
    + 检查App是否使用了默认的TrustManager、SSLSocketFactory、HostnameVerifier实现，如果App使用的实现放宽了认证的要求，则视为不安全
    + 检查SSLErrorHandler的实现是否存在问题
    + 如果WebViews允许使用JavaScript，同时暴露了很多安全、隐私相关的接口，则认为是不安全的。
    + 检查WebViewClient的shouldOverrideUrlLoading()方法是否有实现
    + 检查DexClassLoader等API的使用情况

    检测结果如下表所示，表明OAGs在WebViews方面仍存在很大的安全问题。
![](/images/2019-01-17/5.png)

#### 总结
本文针对OAGs的安全性进行了研究，并表明其中存在的种种安全问题。因为OAGs的特殊性，其造成的影响也很广泛。但文章提出的部分攻击方式在实际中实现所需的前提较强，如需要root等，这一点在文章内容中没有体现。
