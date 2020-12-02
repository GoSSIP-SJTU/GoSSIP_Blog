---
layout: post
title: "Meddling Middlemen: Empirical Analysis of the Risks of Data-Saving Mobile Browsers"
date: 2020-07-07 21:24:05 +0800
comments: true
categories: 
---

> 作者&组织：Brian Kondracki（Stony Brook University）, Assel Aliyeva（Boston University）, Manuel Egele（Boston University）, Jason Polakis（University of Illinois at Chicago）, Nick Nikiforakis（Boston University）
> 
> 会议：[IEEE Symposium on Security and Privacy 2020](https://www.ieee-security.org/TC/SP2020/)
> 
> 链接：[Link](https://www.securitee.org/files/meddlingmiddlemen_oakland2020.pdf)

### Abstract

在本篇论文中，作者对安卓data-saving浏览器的安全和隐私问题进行了全面的研究。

### Introduction

为了带给用户更流畅和快速的体验，在手机浏览器市场中占比更多，许多浏览器开启了”save data“的模式。有这种功能的浏览器被称为data-saving 浏览器（DSB）。该功能通过代理服务器分流用户流量，代理服务器处理应用程序级逻辑，并返回压缩的静态页面和资源。这可以减少用户的网络流量以及客户端设备的计算开销。

在本文中，作者去对DBS生态系统进行了全面的分析。作者首先从Google Playstore中手动选取了9个提供data saving功能的浏览器。然后对这9个浏览器从基础设施、加密、协议头、应用程序行为和用户体验5个维度进行了分析。

- 作者首先进行了对DSB的网络基础进行了探索性调查，并找到了多个网络代理运行着严重过时的软件。
- 作者的实验表明当允许data-saving模式时，网站的功能和行为会有所变化，这将会引入严重的脆弱性。
- 作者建立了一个测试管道，用于比较由web服务器提供的服务与代理优化后到达终端用户设备的数据。
- 此外，作者还强调了data-saving功能还忽略了网站的安全头，破坏了web应用程序的安全保证。

作者的实验表明了在浏览器中启用data-saving模式就相当于通过一个残破的玻璃浏览页面，使得用户在多个维度上受到了不利的影响。

作者的贡献包括

- 作者对流行的浏览器中data-saving功能带来的安全风险进行了首次分析。
- 作者对适用于Android的DSB进行了全面的安全性调查，并揭示了其内部的功能。实验揭示了一系列缺陷，包括错误配置和有问题的操作，这些缺陷会严重影响用户通信的安全性和隐私性。
- 作者负责任得将实验过程提供给浏览器供应商。
<!-- more -->

### Background

#### Data-saving browsers. 

data saving是通过特定的代理服务器代理用户发出的请求，并对响应进行修改和压缩来路由用户的流量完成的。

DSB的核心是快速而强大的代理服务器（通常在物理上更靠近Web服务器），它可以下载完整的资源，潜在地执行所需的逻辑（如脚本），并最终将较小的静态页面返回给用户。通过这种模式，data-saving浏览器供应商承诺为用户节省了超过90%的流量。与典型的HTTP代理或所谓的匿名浏览器相比（他们的目的通常是为了掩盖用户的身份或位置），DSB的目的是为了有效的减少传输中请求的有效负载的大小。作者研究中包括的所有浏览器，都通过X-Forwarded-For HTTP标头使用户的IP地址可以为最终Web服务器使用，因此没有提高对用户的隐私保护。

#### HTTPS interception

为了给用户提供足够的数据节省，Data-saving浏览器需要能够处理HTTPS流量。这也就需要浏览器包含路由HTTPS流量的功能。常规的代理服务器仅需要简单的在源和目的地之间路由HTTPS流量，而data-saving浏览器则需要将端到端的链接分为两个部分。如图一所示。此过程中，诸如URL lock图标之类的视觉提示将保持不变，用户将假定设备和最终web服务器之间的安全通信通道从未中断。

![](/images/2020-07-07/1.PNG)

在图中，可以看到，data-saving的管道引入了其他实体（即中间代理服务器）来处理和修改应用程序和协议级别的信息。因此，关键的服务器定位，附加功能和内容修改的结合使该生态系统成为雷区，其中存在的漏洞可能对数百万用户造成严重的安全威胁和隐私泄露的隐患。

### EXPERIMENTAL SETUP AND METHODOLOGY & EXPERIMENTAL EVALUATION  AND RESULTS

#### A. Proxy Server Identification and Collection

作者选择了如表一所示的9个浏览器。

![](/images/2020-07-07/2.PNG)

#### B. Measurement Infrastructure

实验结构如下图所示。

![](/images/2020-07-07/3.PNG)

作者从web客户端和终端web服务器两个位置进行数据收集——例如，作者使用中间人代理或浏览器的动态检测来记录在用户设备上看到的网络流量；同时，在终端web服务器记录代理池的IP地址，地理位置。

#### C. Quantifying Data Savings

data-saving特性是区分DSB和传统web浏览器的重要因素。据宣传，data-saving模式可以节省用户90%的流量。为了量化该效果，作者在启用和禁用data-saving的两个模式下访问Alexa排名前100位的网站，测量了每个浏览器使用的数据量。作者也量化了DSB在传输视频（YouTube)时提供的流量节省。

##### 实验结果

DBS确实节省了用户的流量。但考虑到HTTPS-by-default的推动，对于不拦截HTTPS流量的浏览器（即Yandex，UC和Ninesky）在流量节省方面的表现很差，原因：由于它们无法/不愿意处理HTTPS流量，因此这两个DSB在数据保存时交换了更多元数据。但其实Ninesky也不处理HTTPS流量，不过作者推断，节省是由于浏览器在客户端阻止了某些资源（例如广告，图像和视频）。

![](/images/2020-07-07/4.PNG)

#### D. Proxy Server Ecosystem Enumeration

由于DSB的服务质量取决于代理服务器基础结构，因此作者量化了每个浏览器供应商的服务器的数量和位置。

为了代理服务器的内部架构，作者使用DSB向他们控制的服务器发出一个web页面的请求，并在服务器端对请求做了记录。作者记录了下面的信息：

- 向作者的服务发送请求的代理服务器的地址（endpointserver  address）；
- 浏览器直连的代理服务器端的地址（gatewayserver  address）；
- 所标识服务器的WHOIS查询结果和IP位置信息。
- 作者还利用NMAP对端点服务器和网关进行指纹识别。

为了更准确的描述DSB的内部框架，作者使用VPN服务更改了设备的地理位置。作者选择了4个国家（美国，德国，日本和巴西）。

##### 实验结果

**Proxy  endpoint  server  enumeration.** 作者发现，Opera mini(Extreme) data-saving模式下使用总共1,481个唯一IP地址连接到作者的服务器，比其他浏览器多出了一到两个数量级，而大多数DSB的IP地址都在几百个范围内。同一供应商浏览器的代理服务器IP地址数量相似。

![](/images/2020-07-07/5.PNG)

为了深入了解DSB生态系统的地理位置，作者根据使用的每个VPN位置分析数据集，对比了HTTP和HTTPS请求的结果，见表III。Opera和UC系列浏览器在作者测试的所有地区都有支持。相反，像Yandex和Puffin这样的浏览器仅有一个单一的代理池，由跨区域的用户共享。

![](/images/2020-07-07/6.PNG)

**Proxy gateway server enumeration.** 图4显示了每个浏览器遇到的代理网关服务器数量。 作者的结果表明网关服务器少于端点服务器，他们的主要作用是实现负载平衡或提高路由效率。 

![](/images/2020-07-07/7.PNG)

**Proxy  Server  Fingerprinting** 表V显示了作者在DSB的网关代理服务器上侦听到的过时软件。作者发现大部分DSB网关代理服务器容易受到各种攻击，如拒绝服务、特权升级和目录遍历。

#### E. ReCAPTCHA

现下最流行的验证码服务reCAPTCHA v3的使用了完全透明的JavaScript“挑战”来替代，而无需用户操作。在这里，作者提出了一个问题：如果用户使用DSB浏览网页，如果共同使用某代理服务器的其他用户表现不佳或被认为是僵尸，他们也会收到较低的reCAPTCHA v3分数吗？（根据reCAPTCHA的官方指南，较低的分数应该触发诸如强制强制认证挑战，电子邮件验证，限制用户行为，或标记潜在的风险交易。）

为了验证DSB是如何影响 reCAPTCHA-保护页面的行为的。作者在每天正常的时间（8am-6pm）周期性得访问从Alexa  Top  1K中选择5-10页面，并选择一组任意的滑动。然后，浏览器被定向到作者控制的页面，作者记录下来reCAPTCHA v3返回的分数。记录请求页面的端点代理的公共IP地址，以及reCAPTCHA v3返回的分数。

##### 实验结果

图5显示了所有浏览器的reCAPTCHA v3分数的分布情况。



![](/images/2020-07-07/9.PNG)

结果表明，用户的浏览经验在每次使用DSB时会发生极大的变化。根据reCAPTCHA v3指南，网站管理员应该将超过0.5的分数视为机器人活动的基准。假设大多数web站点都遵循这个基线，当启用数据保存模式时，DSB用户将会遇到更多的验证码、速率限制甚至IP禁令。

#### F.  Proxy Server Security Auditing

作者考察了在启用和禁用data-saving模式时，TLS密码套件支持和TLS证书报错处理两个方面的安全性是否有所减弱。

##### 实验结果

**Supported Cipher Suites.** 表VI列出了每个浏览器在启用和禁用data-saving的情况下提供的强密码套件和弱密码套件的数量。

所有支持代理HTTPS内容的浏览器都会引入其他弱密码套件，而大多数浏览器也会减少强密码套件的数量。

**SSL  certificate  error  handling.** 表VII总结了关于处理证书错误的测试结果。

作者发现对于大多数错误，Opera和Puffin允许访问并且仅显示警告，而Chrome阻止访问。

Opera浏览器在data-saving模式下接受来自不受信任的证书颁发机构(CAs)的证书。作者为自己控制的域申请了由SuperFish签名的证书，并在data-saving模式下获取了浏览器的信任。结果如下图所示。

#### G. Content Leakage

在本节，作者关注了data-saving是否模式是否会有意或无意的将用户的隐私信息泄露给第三方。作者进行了一系列实验，主要关注了三种类型的敏感数据：访问的URL， 嵌入在WEB页面的数据，以及提供给登录表单的凭据。

**URL leakage** 这里作者URL泄露指的是第三方（代理服务器或者他们选择的DNS解析器）可能会记录用户请求的URL和域，并在以后访问他们。因此为了测试这种类型的泄露，作者访问了他们控制的服务器的URL，由于该URL足够特殊，因此可以断定当对于服务器捕捉到的其他重新访问都可以被认为是泄露。

**Page content leakage** 在实验中，作者创建了许多看起来包含敏感信息的页面在自己的服务器上，并在客户端访问了这些页面，然后检查这些页面是否被再次访问。

**Credential leakage** 作者检查了自己的登录账户是否被非法登录。

#### H. Content Manipulation

为了检查页面被操纵的程度，作者通过DSB访问了他们控制的一系列页面来检测是否有任何内容的修改，他们主要关注了，广告内容的修改和HTTP安全头的修改。作者创建了一组由57个网页返回不同组合的安全敏感的HTTP头。网页还包含多媒体内容和虚假的广告。这可以帮助作者记录由代理添加、修改或删除的任何http头，以及对每个页面的HTML内容所做的任何修改。

##### 实验结果

代理服务器可以自由地修改一个网站的实际HTMLcontent，以及在http响应中用于传输该内容的任何元信息。

**Modifications  to  the  HTTP  transport. ** 测试结果显示DSB代理服务器在HTTP响应中丢失了大量的报头，这使得依赖这些报头的任何安全措施都没有意义。例如，Opera Mini和Opera Browser的代理都不会转发X-Frame-OptionsHTTP标头。实验还发现了HTTP请求中的一系列修改。 特别是，UC Browser, UCMini, Opera, and Opera Mini 收集了大量潜在敏感信息（即IMEI，电话号码，设备序列号和Android广告ID），并将该信息以HTTP标头形式传输到其代理服务器。

**Modifications to HTML content.** 作者没有观察到DSB对HTML内容的任何明显恶意修改。 不过作者观察到 Opera  Mini(High)和Opera对网站内容的修改，代理服务器通过注入CSS内容来阻止广告。 由于隐藏广告可以提高用户的体验，因此作者将该修改归因于良性。

**Leakage.** 作者没有记录到任何第三方试图重新访问之前访问过的页面或使用之前使用过的敏感信息的情况。

### CONCLUSION

本篇论文揭示了主流浏览器的data-saving模式带来的一系列漏洞。作者的发现突显了用户面临的巨大权衡——使用DSB所带来的明显经济利益 & 被**弱TLS密码套件、错误的证书检查、缺乏对安全机制的支持、运行过时软件的代理服务器以及用户可能会被被贴上机器人的标签等问题**之间的权衡。
