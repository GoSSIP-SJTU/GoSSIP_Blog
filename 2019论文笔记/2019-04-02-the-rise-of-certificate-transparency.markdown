---
layout: post
title: "The Rise of Certificate Transparency and Its Implications on the Internet Ecosystem"
date: 2019-04-02 16:37:52 +0800
comments: true
categories: 
---

作者：Quirin Scheitle, Oliver Gasser, Theodor Nolte, Johanna Amann, Lexi Brent, Georg Carle, Ralph Holz, Thomas C. Schmidt, Matthias Wählisch

单位： TUM, HAW Hamburg, ICSI/Corelight/LBNL, The University of Sydney, FU Berlin

原文：[link](https://arxiv.org/pdf/1809.08325)

出处：ACM IMC

<hr/>

#### Introduction

证书公开日志通过公开记录那些CA已经颁发证书，可以为互联网的域名拥有者提供监控自己证书发布情况的平台，意在及时发现恶意的证书颁发行为。

在2018年四月份，Chrome要求所有新颁布的证书都需要记录在证书公开日志中，随着证书公开日志的逐渐普及，一些问题也逐渐显现出来。本篇文章意在研究CT的普及情况，以及其公开透明性所可能产生的负面影响。

<!--more-->

#### CA和CT Log的进化过程

下图描述了从2015年至今CA对CT的支持率，2018年Let‘s Encrypt的加入大大提高了证书的记录率，然而图c中也显示出了问题：Let’s Encrypt只向某些log提交证书，使得这些log负载过大，并且过于单一化的log记录会使得整个系统的稳定性降低。

![fig1](/images/2019-04-02/1.PNG)

#### 服务器对CT的部署情况

##### A. 数据获取

研究者被动监控University of California Berkeley提供的上行线路网络流量，从2017-04-26到2018-05-23，总共获取了 26.5G的TLS连接流量。并且，他们还通过Zmap进行主动扫描了将近423M的域名，以检测服务端的证书部署情况。

##### B. 数据分析

通过分析TLS握手中证书是否包含SCT，以及是通过证书扩展/TLS扩展/OCSP应答中的哪种方式提供的，结果是一共66.76%的服务器响应都包含了SCT。作者手动分析了图中峰值的出现原因，发现是起源于`graph.facebook.com` 这个链接的大量请求，但具体原因不明。

![fig2](/images/2019-04-02/2.PNG)

研究人员还统计了这些证书提供的SCT是由什么log签发的：

![fig3](/images/2019-04-02/3.PNG)

在整个扫描过程中，研究人员发现有16个证书存在SCT验证失败的问题，这些证书分别来自4家CA。经过上报给这些CA，他们确认说这些错误的SCT是来自于最初的试验证书，后面都被修复了。

这些发现还一度引起了一些讨论，就是要不要记录最终的证书。因为CA在签发了证书之后 (这种证书称作precertificate)还需讲其提交到log，经过log签名并将SCT嵌入在证书中形成final certificates，最后颁发给域名拥有者。由于Let‘s Encrypt是将最终证书也记录在CT中的，这又导致了log的性能受到一定程度影响。

#### DNS信息的泄露

在证书的Common Name (CN) 以及Subject Alternatibe Name (SAN)中一般会包含一些子域名，这些统称作 fully qualified domain names (FQDNs)。通过向网上的一些证书公开日志数据库 (如`censys.io、crt.sh`) 发送查询请求，可以获得任何域名的证书信息。因此CT成为了一个可以被利用的子域名发现工具。

研究人员找到的一些常见子域名如下：

![fig4](/images/2019-04-02/4.PNG)

其中值得注意的是webdisk/cpanel/whm都是一些服务器的管理接口，可以尝试对其进行字典攻击。

接着研究人员对获取的子域名进行DNS查询，找到了18.8M个有效的FQDNs。在这些FQDN中，只有1.1M被记录在了Sonar中，因此证明CT为一个有效的子域名发现方案。

#### 发现钓鱼网站

通过对关键字进行搜索，可以找到很多使用了证书的钓鱼网站，文章中例举了其中一些：

![fig5](/images/2019-04-02/5.PNG)

#### CT蜜罐

由于CT存在上述子域名发现功能，研究人员通过故意让CT log记录自己子域名的信息，从而去反向验证是否已经存在上述真实利用行为，发现潜在的攻击者。

研究人员发现，在发布了子域名precertificate的73秒到3分钟之间，其域名就会收到相应的DNS请求。在11个蜜罐中，所有的都收到了来自Google (AS 15169)，1&1 (AS 8560)，Amazon (AS 16509) 和DigitalOcean (AS 14061) 的请求。值得注意的是，域名的泄露途径不只是CT一种，还有其他的比如说FarSight的DNSDB，但研究者在这里说确定泄露的来源就是CT。

![fig6](/images/2019-04-02/6.PNG)

这些域名在随后的几个小时里也接收到了来自另外76个ASes的请求，通过研究那些请求了超过60%的蜜罐点的那些主机，发现了一些已经存在的臭名昭著的恶意第三方扫描服务提供商，以及一些恶意的连接请求和端口扫描。所有这些请求都没有遵守学术界的“Scanning best practice”，比如说提供rDNS，网页和whois信息，因此也就排除了学术研究的可能性。

#### 总结

总而言之，研究调查了目前CT的采用率为大致60%多，另外研究提出了CT的这种透明度所可能带来的隐私泄露风险。通过设置CT蜜罐，证明了CT确实已经被一些恶意的第三方用于子域名发现行为。

其研究思路我在看这篇文章之前已经在考虑，结果发现已经被人研究过了，其中的一些研究方法是值得借鉴的。当然这其中其实还是有些东西值得继续去研究的，证书公开日志所提供的信息中的价值还有待继续挖掘。
