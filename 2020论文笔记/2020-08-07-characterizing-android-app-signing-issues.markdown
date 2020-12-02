---
layout: post
title: "Characterizing Android App Signing Issues"
date: 2020-08-07 11:27:51 +0800
comments: true
categories: 
---

> 会议：ASE’19
> 
> 作者&组织：Haoyu Wang (Beijing University of Posts and Telecommunications)；Hongxuan Liu (Peking University)；Xusheng Xiao (Case Western Reserve University)；Guozhu Meng (SKLOIS, Chinese Academy of Sciences)；cm
> 
> 论文：[Link](https://ieeexplore.ieee.org/document/8952357)

## Introduction

在本篇文章中，作者首次对Android App签名问题进行了大规模研究。

<!-- more -->

## Background

Android使用了自签名证书的机制，即开发者使用自己的证书为App签名，用来签名的证书的私钥有开发者保管，公钥可以被任何人访问。

在Android有三种签名方案：

- **JAR signing scheme** Android最初的版本便有了。

  ![image-20200805110711247](/images/2020-08-07/1.png)

  漏洞：仅验证未压缩的文件的内容，而不是整个APK，存在Janus and Master Key漏洞。

- **APK signing scheme V2（Android 7.0版本以后） & APK signing scheme V3 （Android 9.0版本以后）** V2和V3版本包含了整个APK文件中的所有二进制内容。

## A TAXONOMY OF APP SIGNING ISSUES

作者对Android签名系统的漏洞进行了分类。

![image-20200807131208866](/images/2020-08-07/3.png)

#### App 漏洞

- 用公众已知的私钥签名（Signing Apps with publicly-known Private Keys）。
- META-INF中不受保护的内容（Unprotected Contents in the META-INF）。由于V1仅验证META-INFO外文件的完整性，因此一方面恶意Payload可能会隐藏在之中被加载，另一方面，其中的内容可能会被替换。

#### Exploits

- Compromise the Integrity of APK Files，因为这表明APK已被攻击者修改或打包过程中存在bug。

- Exploiting Master Key Vulnerability，验证和最后安装的不是一个文件。
- Exploiting Janus Vulnerability，攻击者可以将恶意dex文件伪装成APK文件而不会影响其签名。Android运行时会接受APK作为该应用程序早期合法版本的有效更新

-  Exploiting Unsigned Shorts Vulnerability，zip中的文件偏移量应该是未签名的，但被解释为已签名的，从而导致要验证的内容与要执行的内容不同。
- Exploiting Unchecked Name Vulnerability，APK签名验证没有正确地检查名称长度，从而在zip文件的验证方式与它们的提取方式之间产生了差异，从而允许将现有APK中的文件替换为新文件。
- Exploiting the Fake ID Vulnerability，未能正确验证安装证书链。

#### Release Bugs

- V1相关的Bug。

  ![image-20200807131208866](/images/2020-08-07/1.png)

  （1）签名文件与SF文件之间的验证失败；（2）SF文件与MANIFEST.MF之间的验证失败；（3）SF文件不完整或丢失 ，（4）MANIFEST.MF不完整，（5）MANIFEST.MF缺失，（6）MANIFEST.MF和JAR之间的验证失败，（7）没有签名文件，（8）由不同签名组签名的文件和（9） 回滚保护问题（X-Android-APK-Signed属性）

- V2相关的Bug

  ​	对于APK级别的保护，V2签名的APK由四个部分组成，包括（1）ZIP目录的内容，（2）APK签名块，（3）ZIP中央目录和（4）中央目录的ZIP末尾。 这些部分形成保护链。保护链任意部分的缺失都将导致无法安装错误。

- ZIP相关的Bug。

  （1）ZIP末尾的额外字节，（2）无法从zip文件提取某些文件

#### Compatibility Issues

​	摘要算法不兼容。

## STUDY METHODOLOGY

### Dataset

503万个应用，分析了两次以检查研究有签名问题的App是否已经被移除或更新。

### Study Design

- **RQ1: V1和V2版本的签名在野外的分布？**
- **RQ2: 有多少个应用程序面临APK签名问题？**
- **RQ3: App签名问题的演变**

### Methodology

![image-20200807131208866](/images/2020-08-07/4.png)

#### Zip Analysis

​	首先检查Zip末尾搜索中央目录末尾记录（EOCD）来检查是否是一个有效的Zip文件，并检查末尾是否有多余字节（*Extra Byte*）。之后，作者解析了Central Dictionary的每条目录，并且定位了相应的local file header (LFH)，并检查了它们的一致性（*Unchecked Name Vulnerability*）。作者检查了Central Dictionary的偏移和LFG以检查*Exploiting Unsigned Shorts Vulnerability*。作者进一步提取了Zip文件来检查是否*Extraction*。对于*Janus*，作者检查了Zip标题以查看是否存在dex文件填充。对于*Master Key*，作者迭代所有文件并验证是否存在重复的文件名。

#### Manifest Analysis 

​	用以获得该App的基本信息，包括其程序包名称和最低支持的API级别。

#### Verifying V2 Signatures

​	作者首先检查了APK Signing Block并从中提取APK Signature Scheme V2模块（ASSB）以检查签名使用的算法。作者也验证了APK验证链的完整性。

#### Verifying V1 Signatures

​	识别并解析META-INF和MANIFEST.MF（*Unprotected Contents in the META-INF*）。 作者检查了MANIFEST.MF中列出的每个哈希值是否与相应文件一致（ATTACK-2），并检查APK的完整性。 为了识别*Exploiting the Fake ID Vulnerability*，作者使用Android框架修补的方法。作者也检查了V1使用的签名算法。

#### Signature Analysis

​	作者比较了应用使用的签名算法以及现在流行的算法。

## RESULTS AND ANALYSIS

### RQ1 Signing Schemes分布

作者总共分析了2,951,017个不同的APK。 尽管V2方案已被引入1.5年，但仍然有超过93.7％的APK（2,765,752）仍仅使用V1签名方案，而其中只有6.3％的APK（总共185,150）使用V2方案 。另一发现是，某些应用程序由多个证书签名。 有128个应用程序有2个签名，2,003个应用程序有3个签名，15个应用程序有4个签名，而39个应用程序有5个签名。 作者推断可能是App试图使用多个签名来区分应用程序版本或应用程序发布渠道所致。

### RQ2: App Signing Issues

结果表明，签名问题普遍存在于各个市场。

![image-20200807131128094](/images/2020-08-07/5.png)

### Detailed Results

#### Vulnerabilities 

- *用公众已知的私钥签名（Signing Apps with publicly-known Private Keys）*有24个App使用media密钥，746个App使用platform密钥，23619个App使用shared密钥，40985个App使用testkey。

- *META-INF中不受保护的内容（Unprotected Contents in the META-INF）*数据集中有超过553K个应用在META-INF目录中包含某种不受保护的内容。

#### Exploits

​	作者对漏洞分为了6类，但仅在数据集中发现了2类漏洞——4个应用程序（总共安装435次）存在Master Key漏洞。作者将这些应用程序上传到VirusTotal， 结果表明其中有85个被标记为“ Virus：Android.Masterkey”或“ CVE-2013-4787”。表IV列出了按VirusTotal上标记引擎的数量排序后的前3种应用。

![image-20200805163024593](/images/2020-08-07/6.png)

​	作者发现了1,110个应用程序的完整性受到了损害，即MANIFEST.MF中列出的文件丢失。这些应用程序总共下载了71次。

#### Compatibility  Issue

​	作者发现了许多具有兼容性问题的流行App（下载数量达十亿次）。（SHA256WITHRSA在高于18的API级别上被支持）

![image-20200805165009732](/images/2020-08-07/7.png)



#### Release Bugs

​	在腾讯Myapp、小米和OPPO市场上发现了大量含有Zip bug的应用，这表明这些市场没有严格监管（因为这些应用是低质量的，不能安装在任何Android设备上）。

### App签名问题的分布

​	作者在这里分析了app签名问题与App种类和App流行程度的相关性。

- **App种类**

  ​	![image-20200806100841561](/images/2020-08-07/8.png)

  大多数具有签名问题的App属于游戏、生活、个性化和教育等类别。

- **App流行度**

  ![image-20200806101136701](/images/2020-08-07/9.png)、

​	可以发现有相当数量的App具有百万数量的下载量。

## The Evolution of Signing Issues

作者考察了有签名问题的App会在应用市场上停留多久，于是在七个月后又重新测试了这些App。

####  Release/Update  Time

![image-20200806102031130](/images/2020-08-07/10.png)

​		超过一半的漏洞和攻击是在2016年之前发布的，这意味着它们已经影响了数百万用户至少两年。该结果表明，一方面，这些应用程序的开发人员通常很少关注签名问题；另一方面，应用程序市场没有对应用程序的签名问题进行检查。

####  Post Analysis

​	图7显示了有签名问题的App的剩余量百分比和已经被移动（下架/更新）的百分比。作者发现，不同市场的应用程序监管和应用程序维护行为存在显著差异。

​	![image-20200806102513152](/images/2020-08-07/11.png)