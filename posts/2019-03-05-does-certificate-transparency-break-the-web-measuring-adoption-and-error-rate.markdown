---
layout: post
title: "Does Certificate Transparency Break the Web? Measuring Adoption and Error Rate"
date: 2019-03-05 16:07:02 +0800
comments: true
categories: 
---

作者: Emily Stark, Ryan Sleevi, Rijad Muminovic, Devon O’Brien, Eran Messeri,
Adrienne Porter Felt, Brendan McMillion, Parisa Tabriz  

单位: Google, University of Sarajevo, Cloudflare

出处: IEEE Symposium on Security & Privacy 2019

原文: [pdf](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/314fca4308f1dd1faeeb975bf25f6904af0264f9.pdf)


## Abstract

Certificate Transparency (CT) 是一种近年来提出的证书日志记录方案，作为公钥基础设施的一部分，通过记录Certificate Authority (CA) 公开颁发的数字证书信息，以快速找到恶意或者错误颁发的证书。尽管CT被越来越多的浏览器所接受，想要将其进行全面部署依旧是具有挑战性的任务，其中可能会涉及到兼容性以及用户体验的问题。本篇文章通过讨论CT设计架构的特性，分析其中可能存在的安全问题和缺陷，来向我们展示如何在浏览器端更好地部署大范围的安全改进措施。


<!--more-->

## Introduction

#### A. HTTPS and web PKI

HTTPS提供了对网络流量进行加密以及验证的功能，当用户通过TCP连接网站的服务器之时，还需要服务器提供额外的验证信息，也就是数字证书，来证明其身份的有效性。网站的证书由公认的CA颁发，每个用户的浏览器中维护一个*trust store*，以信任那些由*trust store*中CA颁发的证书。当一个CA被恶意攻击者控制时，可能会自行签发任意证书，这将可能导致一些网站遭受中间人攻击。

#### B. Certificate Transparency

CT通过保证将所有的证书发布消息都记录在公开审计的、只可追加的日志中，以及时发现可疑的证书颁发行为。域名拥有者可以自行对证书颁发日志进行审计，当发现有恶意的证书发布时，可以请求CA及时对证书进行撤销，或者找出存在问题的CA，并通知浏览器将其从*trust store*中移除。CT机制可以从以下三个方面描述：

+ Logging: 无论是CA自身或者是第三方 (扫描器、爬虫、域名所有者等)，都可以向CT log提交证书，当log收到证书后会用私钥签名，并返回一个Signed Certificate Timestamp (SCT)。
+ SCT validation: 当证书携带有效的SCT时，就意味着此证书已经被CT log记录下来了。SCT可以选择由以下方式给出：嵌入在证书中，TLS extension，OCSP stapling。
+ Monitoring/auditing: CT logs提供了一个REST API，可以让任何人去监控这些已经被记录下的证书，并审计这个日志是否正常运作。审计方可以向日志提交*inclusion proof*请求，以保证给定SCT的证书确实被记录在此日志中。也可以提交*consistency proof*来验证其可追加性。

#### C. CT deployment in browsers

目前而言，只有Chrome对CT的支持最好，此文章主要讨论Chrome的CT部署政策。Chrome需要证书拥有2~5个不同CT log签发的SCTs，因此对于恶意证书而言需要同时控制多个CT log才能够欺骗浏览器，使得攻击难度增加。当证书不满足要求时，Chrome会向用户提示如下内容：

![fig1](/images/2019-03-05/1.png)

## Methodology

本篇通过收集分析Chrome usage metrics program提供的数据，考察了CT的采用率以及失误率。研究人员还收集了CT相关的证书错误报警数据，并通过Googlebot爬虫获取了各网站对CT的采用情况。

研究中所分析的网站来源如下：Alexa Top 1000、Chrome User Experience Report、HTTP Archive。

另外，作者还分析了CT log的运作模式，以及Chrome product help forums上与CT有关的贴子，以反映用户对相关错误的反应情况。

![fig2](/images/2019-03-05/2.png)

## Result

#### A. Adoption

下表中列出了采用CT的百分比，以及采用的原因。

![fig3](/images/2019-03-05/3.png)

#### B. Compliance

当网站开发者或者CA在CT相关的配置上出错时，可能导致浏览器做出以下行为：(1) 中断连接；(2) EV 降级。前者可能由于网站使用的CA在浏览器安全政策要求中必须采用CT，或者是网站有*EXPECT-CT*头部。而当EV证书不符合浏览器要求时，Chrome会移除其EV的UI。下表展示了研究人员发现的CT错误原因：

![fig4](/images/2019-03-05/4.png)

#### C. User Impact

根据CT的设计，SCT验证的过程所带来的网站浏览延迟是比较小的，在2018 9月24结束的调查显示，99%的SCT验证在13.3ms之内完成，平均耗时1.9ms。

然而接近50%的用户对于浏览器的错误提示选择忽视，继续照常访问带有错误证书的网站。并且在用户论坛的贴子中，很多人抱怨证书CT相关配置错误问题得不到解决，很多人干脆选择更换浏览器。因此研究人员提出，为了用户浏览的安全性，各大浏览器须尽快部署对CT的支持。

#### D. Risks

研究者从两个方面探讨了部署CT证书不当可能产生的问题：

+ Log distrust: 当CT log不满足浏览器的某些安全要求时，有可能被移除信任证书日志列表。当这种情况发生时，网站可能需要更换其在TLS extension或者OCSP response中所提供的SCT，或者由于证书本身嵌入了SCT，整个证书需要被替代。下图表5中展示了若某个log被浏览器取消信任，会影响到的网站的数量。
+ Server-side SCT delivery: 若网站开发者选择通过TLS extension或者OCSP来发送SCT，则好处是可以及时响应log distrust并更换SCT；麻烦的地方在于必须确保网站是从最新的log信息中提取的SCT，并且由于这种在服务端提供SCT的方式 (在Nginx或者Apache中提供) 目前处于初级的非商业阶段，并没有一套很完善的方法，因此可能在实践中会碰到更多安全问题。

![fig5](/images/2019-03-05/5.png)

## Discussion

本文的作者建议采取分阶段部署的方式，以防止对用户体验产生负面影响，同时呼吁主流的浏览器积极部署CT，以进一步促进其采用率。同时，值得将来继续研究的点包括：网站服务器端SCT  delivery方案机制的完善，当log被取消信任时各方如何应对，以及如何尽量在不提高浏览器的报警率的前提下平缓地全面部署CT等等。