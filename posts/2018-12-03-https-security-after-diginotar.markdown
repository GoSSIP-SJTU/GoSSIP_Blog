---
layout: post
title: "Mission Accomplished? HTTPS Security after DigiNotar"
date: 2018-12-03 16:01:10 +0800
comments: true
categories: 
---

作者：Johanna Amann, Oliver Gasser, Quirin Scheitle, Lexi Brent, Georg Carle, Ralph Holz

单位：ICSI / LBL / Corelight, Technical University of Munich, The University of Sydney

出处：IMC 17  

原文：http://conferences.sigcomm.org/imc/2017/papers/imc17-final227.pdf

##简介

为了防范各种针对SSL/TLS协议设计和实现的攻击，TLS、HTTPS、web PKI增加了很多新的特性。
这篇文章的主要工作就是对这些新特性的使用情况作了一个大规模调查。

 - 作者主要从不同新特性的应用普遍性、各种特性部署的正确性、特性应用普遍性与部署难度之间的关系特性部署的相关性等多个维度进行调查。
 - 考察的安全特性主要包括：Certificate Transparency(CT)，HSTS，HPKP，CAA，TLSA，SCSV等。
 - 作者声称这是第一个针对这些特性应用情况的大规模调查（实际上不是，NDSS'15上有篇文章是针对HSTS、HPKP部署情况的调查，只是调查的范围没有这次广、覆盖的安全特性没有这次多，其它也有一些相关工作提及部分安全特性的应用情况）。
 - 作者采用了主动扫描和被动测量的方法，调查覆盖的域名数量达193M，是截止到2017.3所有注册域名数量(330.6M)的58%。

<!--more-->

**补充说明**：

 - CT：Google主推的一项透明审查技术，主要目的是提供一个开放的审计监控系统，可以让域名所有者确定证书是否被错误签发或恶意使用，以缓解现有SSL/TLS协议信任模型中对CA过度信任的问题。
 - HSTS:HTTP Strict Transport Security，强制客户端使用HTTPs与服务器连接，可以很大程度防范SSL Stripping攻击。
 - HPKP：HTTP Public Key Pinning，主要是防止其它可信CA未经网络拥有者授权为网站颁发证书。对防止攻击者攻破CA恶意签发证书进行中间人攻击比较有效，典型的案例是DigiNotar事件。**值得一提的是，2017年10月，大力推广该技术的Google宣布不再支持HPKP，并计划在2018年5月的Chrome 67解除对HPKP的支持**。
 - CAA：Certificate Authority Authorization，基于DNS的一项扩展，域名所有者在其域名记录的CAA字段中，授权指定CA为其域名签发证书，弥补SSL/TLS任意CA能够为任意域名颁发证书的不足。
 - TLSA：又称DANE-TLSA，DNS-Based Authentication of Named Entities TLS，使用DNSSEC基础设施来保存TLS协议中用到的数字证书或公钥，依托DNSSEC基础设施来限制TLS服务器可用的CA范围。TLSA是一种DNS资源记录的名称。
 - SCSV：SCSV扩展，主要被用来防止协议回滚降级到低版本，是用来避免POODLE攻击的一种方法。


### 研究方法
作者采用了主动扫描和被动测量相结合的方法，将主动扫描得到的网络流量保存到一个pcap trace，用相同的分析方法来处理主动扫描和被动测量得到的数据。

不同特性数据的获取：

 - CT：从X.509证书、TLS及OCSP扩展里提取SCT（Signed Certificate Timestamps）信息。使用了一个修改后的Google log监视软件来从Chrome接受的log中获取证书。
 - HSTS/HPKP：解析扫描器收集到的HTTP response。
 - SCSV：降低TLS版本，并设置Signaling Cipher Suite Value来进行降级保护。这是客户端应该拒绝链接。
 - CAA/TLSA：从DNS收集这些资源记录。


主动扫描：

 - 在悉尼大学（IPv4）和慕尼黑工业大学（IPv4 & IPv6）进行主动扫描； 
 - 扫描是基于域名的而不是IP地址的，这样的好处是可以处理基于SNI的服务器（多域名复用一个IP地址）。
 - 扫描域名的策略是把之前相关工作扫描的根域名取并集，最终收集到了193M个域名。

![](/images/2018-12-03/2.png)

被动监测：

 - 为了分析Certificate Transparency的使用情况，监测了伯克利的Internet uplink几周；同时在悉尼大学和慕尼黑工业大学监测了两周来验证UCB的监测结果。
 - 为了分析TLS版本演进，使用了ICSI的SSL Notary的数据。

![](/images/2018-12-03/3.png)


### 调查结果

#### Certificate Transparency
![](/images/2018-12-03/4.png)

 - 如果一个域名的任意一个IP地址的SSL链接传输了SCT，就认为它是支持CT的。
 - 在悉尼大学和慕尼黑大学的扫描结果相近，12.7%-13.3%的TLS链接中有SCT，差不多是有 6.8M的域名支持Certificate Transparency技术。IPv6支持CT的有357K个域名，这个和IPv6部署的少有关。

![](/images/2018-12-03/5.png)

上图是流行网站的CT技术应用情况。

 - 作者发现越流行的网站利用TLS扩展传送SCT的就越多，考虑到通过TLS扩展传送SCT仅在客户端要求的情况下才会发生，作者推测这是因为流行网络需要优化移动端体验，不把SCT包含在证书里可以在mobile HTTPS事务开始时少传输数百个字节左右。
 - 作者的数据集分析发现，大部分的SCT都是嵌入在X.509证书里的，只有不到1000个在TLS扩展里，49个在OCSP staple里。
 - 几乎所有支持CT的域名都提供一个来自Google管理的log里的SCT，以及一个来自非Google log的SCT，这是Chrome对EV证书的最低要求。
 - UCB收集的数据集里，SCT的出现频率高一点，进一步分析发现56%的支持用TLS扩展传送SCT的域名是属于Google的，基本上都是流行网站在使用CT技术。
 - UCB收集的数据中，74.311个（99.2%）嵌入了SCT的证书是在443端口上的，279个在80端口上。

CA和嵌入SCT的证书的关系：

 - 少数CA签发了大多数嵌入了SCT的证书：Symantec签发了67.16%的嵌入SCT的证书。可能是因为Symantec之前的误签发事故，Google要求Symantec记录它签发的所有证书。
 - 其它CA包括GlobalSign，Commodo，StartCom，其中StartCom和其父公司WoSign新签发的证书已经不被Mozilla信任了，Chrome在2017年9月也不再信任它们的证书。

关于SCT的分析结论是：

 - 还有很多CA对提供嵌入式SCT没什么兴趣，换句话说是不打算支持CT，这个情况从2014年至今没有变化；
 - Google的计划是证书需要被多个Log维护者记录log，但是大多数证书只被Google记录了，certificate Transparency技术的推广情况和Google的预期不太一致，不是很理想。
 - 还有部分证书包含了错误的SCT。

#### HSTS 和 HPKP
作者主要从部署、一致性（是不是不同IP返回的HSTS/HPKP header一致）、生命周期、密码学合法性等几个方面来考察。
![](/images/2018-12-03/6.png)

 - 极少数域名存在header不一致性；
 - 约3.5%的域名支持HSTS，0.02%的支持HPKP（低普及率也是Google停止支持HPKP的主要原因之一）。
 - 支持HSTS和HPKP的域名有一些存在部署错误，一般是在设置max-age，includeSubDomains等问题上出错。
 - 流行域名对HSTS和HPKP的支持情况比较好。

![](/images/2018-12-03/7.png)


#### SCSV
![](/images/2018-12-03/8.png)

 - SCSV是支持的最多的一项技术。主要原因是主流密码学库里提供了SCSV的支持。
 - 部分流行网站不支持SCSV，因为它们使用了IIS，而IIS和SChannel不支持SCSV。考虑到IIS占用了11%的HTTP服务器市场份额，影响还挺大的。


#### CAA及TLSA
![](/images/2018-12-03/9.png)

 - CAA推广的时间比较晚，但是2017年的扫描结果和2016的一项工作对比，部署的增长比较好，有望进一步推广，有意思的是很多CAA记录里的issue属性指定的是Let's Encrypt.
 - TLSA是依托DNSSEC的，而DNSSEC部署比较少，所以TLSA的应用情况也不太乐观。支持TLSA的域名中验证DNSSEC签名的比率明显比支持CAA的高很多。

#### TLS版本演进

 - 2013年左右应用最多的是SSL v3和TLS 1.0.
 - 2014年底之后应用的较多的就是TLS 1.2了，TLS 1.1被用的比较少，主要是因为OpenSSL同时支持了TLS 1.1和TLS 1.2，很多网站直接从TLS 1.0跳到了TLS 1.2.
 - 作者一共观察到700万个TLS 1.3链接，峰值是2017年2月Google Chrome 56支持TLS 1.3的时候出现的，但Chrome停用TLS 1.3后又降下去了。

#### 总结
从文章的调查可以看出来，尽管学术界和工业界提出了多种方案来帮助防范各种针对SSL/TLS协议的攻击，这些方案的实际应用非常少，Google主推的Certificate Transparency和HPKP技术应用情况都不理想。网站管理者们大多倾向于采用部署难度低的方案，比如密码学库直接支持的SCSV，这也侧面反映了密码学库的安全性对整个SSL/TLS生态环境安全性的重要性；而需要网站管理者自己配置的方案常常会出现错误，比如配置HSTS和HPKP。

2018年3月，IETF正式批准TLS 1.3成为互联网标准，全面升级到TLS 1.3是应对现有针对TLS攻击的一个好的方案。
