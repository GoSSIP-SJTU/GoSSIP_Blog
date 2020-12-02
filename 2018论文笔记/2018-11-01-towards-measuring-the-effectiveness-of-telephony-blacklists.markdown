---
layout: post
title: "Towards Measuring the Effectiveness of Telephony Blacklists"
date: 2018-11-01 19:32:50 +0800
comments: true
categories: 
---
> 作者：Sharbani Pandit, Roberto Perdisci, Mustaque Ahamad, Payas Gupta
>
> 单位：Georgia Institute of Technology, University of Georgia, Pindrop
>
> 出处：NDSS'18
>
> 原文链接： http://cyber.gatech.edu/sites/default/files/images/towards_measuring_the_effectiveness_of_telephony_blacklists.pdf

<hr/>

## 背景

本文是美国佐治亚理工学院的研究人员发表在NDSS18上的研究。这篇论文研究了如何有效地阻拦恶意电话。现在有很多恶意电话会利用一些网络技术 （机器人呼叫，声音模仿，号码欺骗等等）对手机用户进行骚扰或诈骗。常见的防护方法就是建立黑名单。目前也有很多APP或设备安全服务在建立这些黑名单。但是具体如何建立黑名单，黑名单效果怎么样都是未公开的。所以这篇文章通过自己收集数据，并利用不同类型数据建立多种黑名单并评估，进而分析黑名单的效果。

<!--more-->

## 数据收集：

收集恶意号码的途径主要有两种：群众举报，通过大量测试号码（honeypot）等待被动攻击。再根据收集数据是否包含详细通话内容，将所有数据可以分为四类：
### Context-Less Data:
1）FTC dataset (FTC)：群众在美国联邦贸易委员会举报的号码，出于隐私保护，这里只能拿到恶意号码和时间，没有其它信息。
共1.56 million 条举报，300,000 不同号码 （February to June 2016）。
2）Honeypot call detail records (CDR)：honeypot接到的电话，同样包含来电号码，honeypot号码，及时间信息。
共200,000 恶意号码，58,000 目标号码（honeypot号码）（February to June 2016）。
###Context-Rich Data:
Crowd-sourced online complaints (COC): 一些三方网站（800notes.com, MrNumber.com等）接到的举报，包含具体内容文字描述。
600,000条举报（Dec 1, 2015 and May 20, 2016）
Honeypot call transcripts (HCT)：honeypot接到的电话，包含录音记录。
19,090 语音记录， 9,434 恶意号码（February 17, 2016 to May 31, 2016）

以下是具体数据分析：
![](/images/2018-11-01/2.png)

每个dataset数据量的时间分布：周期波谷是因为周末会减少。断档是因为一些原因导致某段时间没有收集数据。

![](/images/2018-11-01/3.png)

相同号码的数据量，有很大一部分号码只被举报几次或被honeypot检测到几次。

![](/images/2018-11-01/4.png)

每个号码被honeypot检测到与被举报的时间差，大部分几乎是同时。

![](/images/2018-11-01/5.png)

##建立黑名单：
###Context-less blacklisting:
1）blacklist using CDR (honeypot接到的电话): 
可能存在的噪声：误拨电话
建立黑名单过程：过滤，首先根据number of calls 和 number of destination honey pot numbers， 来过滤次数较少的号码；评分：同样根据单位时间内这两项数据来计算评分。阈值：根据CDR与FTC数据交集内号码的评分来决定阈值。


2）blacklist using FTC (官方举报号码):
可能存在的噪声：举报时打错字
根据举报数量过滤次数少的号码。

3）blacklist using COCNC (有内容的三方网站举报)：
这里忽略内容描述，和FTC类似。

###Context-rich blacklisting：
4）blacklist using HTC (honeypot接到的带录音的电话):
建立黑名单过程：主题：首先通过语音分析提取关键词，定义恶意主题; 分类：根据计算权重值，决定每个号码是否属于某个主题。 最后根据这个权重值决定是否列入黑名单。

![](/images/2018-11-01/6.png)

5）blacklist using COC (有内容的三方网站举报):
首先将次数少的号码去掉。
之后和HTC类型，先确定主题。这里是文字解析，会比语音复杂 （语音大部分是机器人固定语音，文字更多拼写错误，且描述太简略）。

![](/images/2018-11-01/7.png)

##评估
###黑名单数据分析：
举报和被动攻击的号码差别较大。因为 honey pot 的号码很多是被弃用的企业号码，而举报的被攻击号码更多是个人号码。因此恶意号码会不太相同。
​                            context-less的黑名单较多，因为有内容的需要根据内容来区分，这个主题匹配比较严格。


![](/images/2018-11-01/8.png)

###黑名单可用性分析：
这里利用两个三方的黑名单来评估
Youmail检测数恶意的号码87%都在作者的黑名单中，遗漏的号码在FTC中，但是举报次数太少被过滤。如果作者不过滤，两个黑名单有98%相似度。
Truecaller检测数恶意的号码只有13%在作者的黑名单中，因为Truecaller只要接到一次举报就加入黑名单，如果只考虑Truecaller接到举报次数大于5的号码，其中75%都包含在作者黑名单中。
这说明作者的黑名单和现实的黑名单类似，因此可以用来评估现实中黑名单的效果。
作者还利用Whitepages分析黑名单中号码来源：

![](/images/2018-11-01/9.png)

VoIP利用网络IP打电话，toll-free免费的机构服务电话，landline固定线路
大部分号码没有用户信息，5%没有运营商，67%是VoIP。
Whitepages只认为22%号码是恶意的，说明Whitepages的恶意检测效果不是很好。

###黑名单效果分析：
每隔一天做一次实验，用这一天之前的数据建立黑名单，来拦截接下来一天的电话，计算每天的拦截率。
Context-less blacklisting拦截率高于Context-rich blacklisting

![](/images/2018-11-01/10.png)

![](/images/2018-11-01/11.png)

![](/images/2018-11-01/12.png)

![](/images/2018-11-01/13.png)



###误报率分析：
自己准备白名单，100,000 phone numbers listed across 15 different major US cities and 10 different business categories
只有10个被包含进了黑名单，误报率很低。

### 其它
最后作者还简单分析了下，相同主题可能出现在不同的数据库中，相同的号码也可能用来不同的主题。
