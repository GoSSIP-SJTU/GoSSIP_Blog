---
layout: post
title: "Why Does Your Data Leak? Uncovering the Data Leakage in Cloud from Mobile Apps"
date: 2019-03-19 15:35:16 +0800
comments: true
categories: 
---

作者：Chaoshun Zuo, Zhiqiang Lin, Yinqian Zhang

单位：The Ohio State University

出处：S&P 2019  

原文链接：http://web.cse.ohio-state.edu/~lin.3021/file/SP19.pdf

<hr/>
#### 概述
越来越多的应用使用云服务商提供的APIs(Cloud APIs)实现数据存储、数据分析、消息推送等功能。然而，App存储在云端的数据经常出现泄露现象。在文章中，作者从App中错误地使用云端APIs这一角度出发，设计了并实现了LeakScope这一工具，对Google Play上160万个App进行自动化的分析，希望在App中发现因为误用云端API所导致的云端数据泄露漏洞。最终，作者发现使用Amazon、Google和Microsoft云服务的15098个App都存在此种类型的问题，并且已与相关App开发者联系，对问题进行修复。

<!--more-->
#### Cloud APIs
在云服务商提供Cloud APIs之前，App开发人员需要同时维护前端的App和后端的服务器，并保证服务端的安全性、稳定性、高可用性等，这对开发人员有较高的要求。因而主流的云服务厂商包括Amazon、Google、Microsoft等推出了mBaaS(mobile back-end as a service)，包括App开发所需的APIs、管理平台等，将后端的细节进行封装。开发人员仅仅需要简单的配置，便可快速地构建起App的后端，并依赖云厂商提供安全性、可靠性、可扩展性的保证。通常云服务商提供下图所示的四个模块的功能，相关的APIs包括setValue、createUserWithEmailAndPassword、sendPasswordResetEmail等。
![](/images/2019-03-19/1.png)

##### 如何使用Cloud APIs
在使用Cloud APIs时，云端需要知道是哪一App在调用相关的API，具体涉及到两个密钥(Key)
+ App Key
    权限非常有限，仅能访问一些公开资源。云端利用该Key来确定是哪一App在调用相关API。因为多个App可能使用同一云端数据库、存储空间，所以云服务商利用App Key将这些资源虚拟化、隔离。
+ Root Key
    权限很大，可以访问App所有后端资源。因而开发者应当将该密钥保密，只有系统管理人员可以使用。

下图所示为App Key和Root Key的使用示例。
![](/images/2019-03-19/2.png)

#### 数据泄露漏洞
##### 认证时Key的误用
App在使用Cloud APIs时需要将Key传到云端，这样云端才能够将App和相应的虚拟化后端服务器联系起来。但往往开发者忽略了App Key和Root Key之间的区别，在应当使用App Key的地方错误地使用了Root Key，因而导致漏洞的出现。作者发现使用亚马逊、微软云服务的App存在相关的问题，包括微软Azure存储、Azuer通知中心、Amazon AWS S3。
以Azure存储服务器为例，该云服务为开发人员提供了两种Key:
+ Account Key
    类似上文中所述的Root Key，可以任意读取存储服务器上的内容。
+ Shared Access Signature(SAS)
    开发者可以通过配置权限的方式来限制该Key可以访问的数据。
很显然，开发者在App中应当仅使用SAS，并将Account Key保密。然而，作者通过实验发现，如下图示例所示，许多开发者都没有遵守这一规范，在App中硬编码Account Key或存在asset文件中，攻击者可以轻松地通过逆向App获取。

![](/images/2019-03-19/3.png)

##### 授权时用户权限的错误配置
因为配置权限方法的多样性等原因，开发者有可能没有正确配置权限。作者分别在使用Firebase和AWS作为后端的App中发现配置问题。
+ 未做身份认证(任何人都可以访问)
+ 未做权限检查(所有经过身份认证的用户均可以访问)
![](/images/2019-03-19/4.png)source/_posts/2019-03-19-why-does-your-data-leak-uncovering-the-data-leakage-in-cloud-from-mobile-apps.markdown


#### LeakScope设计与实现
目标功能：
+ 识别出App中的Cloud APIs和Key相关字符串(考虑混淆的情况)
+ 在不泄露数据的条件下证明存在数据泄露问题

![](/images/2019-03-19/5.png)

source/_posts/2019-03-19-why-does-your-data-leak-uncovering-the-data-leakage-in-cloud-from-mobile-apps.markdown
##### 功能模块：
+ Cloud APIs识别
    首先，作者收集了Amazon、Google、Microsoft云服务的常用API列表(32个)。为了处理混淆的情况，作者为每一函数生成签名，包括SDK中的Cloud APIs，并进行搜索匹配。生成签名的策略在于关注混淆前后的不变量：分层结构和函数内部调用关系。
    ![](/images/2019-03-19/7.png)

    为函数生成签名的策略(标准化)：

    ![](/images/2019-03-19/8.png)

+ 通过分析字符串获取Kesource/_posts/2019-03-19-why-does-your-data-leak-uncovering-the-data-leakage-in-cloud-from-mobile-apps.markdowny的值
    首先，生成跨函数的控制流图(CFG)，并以Cloud APIs的参数为起点向上回溯，讲与该参数有关的变量加入字符串栈中，同时加入到data dependence graph(DDG)中。如果存在函数调用，则递归地回溯调用的函数。在回溯结束后，从字符串栈顶开始，结合CFG、DDG“执行”与字符串变量相关的操作，最终得到Cloud APIs所使用的参数的值。
+ 验证漏洞存在
    + 一些云服务商的App Key和Root Key格式不同，因而可以相对简单地识别
    + AWS的App Key和Root Key格式相同，因而需要设计出在不泄露数据的情况下验证存在数据泄露问题的方案。作者发现，以App Key和Root Key向服务器请求不存在的数据，返回结果不同，分别为“permission denied”和“data not found”，通过该方式可以成功进行验证。
    + 针对权限配置错误问题，作者对Firebase的REST API进行测试，同时也对AWS的IAM进行测试。
##### 实现
基于dexlib2实现Cloud APIs识别功能，基于Soot实现字符串分析功能。验证漏洞通过Python脚本发送Request、解析Response。整个工具代码量在6000行左右。

#### 评估
+ 实验环境：
    + Intel Xeon E5-2640 CPU 
    + 24核 
    + 96G内存
    + 运行Ubuntu 16.04操作系统
+ App收集
    + 使用爬虫脚本在两个月收集到Google Play上的1609983个免费App
    + 占用空间15.42TB
+ 实验结果
    + 6894.89 CPU小时
    + 检测出15098个App中的17299个漏洞
+ Cloud APIs识别：超过90%的App使用Firebase的Cloud API
    ![](/images/2019-03-19/6.png)
+ String Value Analysis: 存在无法分析出的字符串
    ![](/images/2019-03-19/9.png)
+ Vulnerability Analysis
    ![](/images/2019-03-19/10.png)
+ 分析
    + 严重性
        有问题的App中，10个下载量在1亿到5亿，14个下载量在5000万到1亿。
    + 混淆VS未混淆
        即使经过混淆，作者的工具也可以识别出问题。
    + 误报分析
        有些开发者可能故意将一些资源公开，作者对30个有问题的App，发现其中86.7%都是存在泄露用户照片、上传图片等问题。



#### 总结
文章针对Android App进行大批量分析，这一类文章的结构比较类似。同类型的文章还有不少，如作者的另外一篇《Authscope: Towards automatic discovery of vulnerable authorizations in online services》。对于权限配置错误的描述不够具体。