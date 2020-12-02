---
layout: post
title: "An End-to-End Large-Scale Measurement of DNS-over-Encryption: How Far Have We Come?"
date: 2019-12-10 23:11:20 -0500
comments: true
categories: 
---
> 作者：Chaoyi Lu, Baojun Liu, Zhou Li, Shuang Hao, Haixin Duan, Mingming Zhang, Chunying Leng, Ying Liu, Zaifeng Zhang and Jianping Wu.
>
> 单位：Tsinghua University, University of California, Irvine, University of Texas at Dallas, Qi An Xin Technology Research Institute, 360 Netlab.
>
> 出处：*IMC’19*
>
> 原文：[link](http://delivery.acm.org/10.1145/3360000/3355580/p22-Lu.pdf)

---

### Abstract

DNS原本是以明文包的方式发送，但为了保护DNS数据不被攻击者篡改，近几年逐渐产生了一些用于加密server端和client端DNS数据的协议，可以称作DNS-over-Encryption (DoE)。本文首次对DoE进行了大规模端到端的研究。通过互联网扫描、用户端测量以及被动流量分析，研究发现对于DNS client而言，使用DoE可以在不带来过度延迟的同时防御in-path劫持攻击；但同时作者也发现一些问题，比如有25%的DNS-over-TLS服务提供的是无效证书。虽然目前DoE的使用率还比较少，但存在上升趋势，作者倡导更广泛地部署此项技术，并同时关注在部署过程中可能出现的问题。

<!--more-->

### 1 Introduction

DNS作为互联网中根本的building blocks之一，提供了从域名到IP地址的映射。然而，在最初IETF标准中规定DNS包通关UDP协议明文传输，这就给恶意攻击着提供了可乘之机。在现实世界里已经存在了监控、截取DNS数据流的攻击，如NSA的MoreCowBell和QuantumDNS。另外，有研究证明网络中的中见组件 (middleboxes) 会对DNS进行流量劫持与篡改。

本文从如下几个问题着手，对DoE的部署现状进行了研究：

+ 1) 有多少网络提供上部署了DoE？部署是否安全。
+ 2) 从用户角度而言，DoE带来的性能表现如何？是否存在由DoE导致网站无法访问的情况。
+ 3) 真实世界中的DoE是什么样的？

### 2 Background

![](/images/2019-12-10/1.PNG)



作为主流的防御手段，目前DNS加密有五种方式：DNS-over-TLS (DoT), DNS-over-HTTPS (DoH), DNS-over-DTLS, DNS-over-QUIC以及DNSCrypt，在文中将他们统称为DoE。研究人员对上面提到的五种DoE方法从如下几个方面进行了评估总结：

+ 协议设计：1) 新协议是基于传统DNS协议还是利用了新协议；2) 是否提供了fallback选项。
+ 安全性：1) 是否给予标准密码学协议 (e.g., TLS)；2) 是否能防御on-path被动DNS流量分析。
+ 可用性：1) 是否需要改变软件配置；2) 是否产生overhead。
+ 可部署性：1) 新协议是否基于标准协议；2) 是否被主流DNS软件支持 (e.g., BIND)。
+ 成熟性：1) 新协议是否已被IETF标准化；2) 是否被DNS服务提供商广泛支持 (e.g., 公共DNS解析服务)。

![](/images/2019-12-10/2.PNG)

#### 1) DoT

DoT于2016年发表与RFC7858，基本思想是：client和server协商TLS会话之后再进行DNS查询，将DNS消息通过TCP传输。因此client与DNS递归解析服务器可以进行DNS的加密传输，同时client还可以通过SSL证书来验证服务器的身份。但由于DoT使用的是固定端口853，因此易被流量分析识别。

DoT广泛地被各个OS系统以及DNS软件所支持，然而，client想要使用DoT必须手动进行配置比如说更换stub resolvers (e.g., 更新OS或者安装stub resolver比如Stubby) 并手动设置DoT解析服务器。由于DoT提供的连接重用功能，其可能带来的延迟会被减轻。

#### 2) DNS-over-DTLS

作为DoT的变种，DNS-over-DTLS使用UDP以带来更好的性能。然而它只是作为DoT的备用协议，因此在现实场景中并没有出现相关应用。

#### 3) DoH

DoH在RFC8484中提出，使用URI模板 (e.g., `https://dns.example.com/dns-query{?dns}`) 来查询服务器的IP，其中DNS包被编码在URI的GET参数或者POST数据中。下图为DoH的两种请求形式：

![](/images/2019-12-10/3.PNG)

由于DoH重复使用了HTTPS的443端口，因此可以保证DNS流量不被区分出来。并且DoH会对DNS服务器进行加密认证，不提供fallback选项。由于DoH是运行在HTTPS之上，它非常适合被部署在web浏览器端，并且不需要client进行手动的更改。但对于DNS解析服务器，需要手动部署支持DoH，目前Cloudflare、Google以及Quad9提供的大型公开DNS解析服务器可以支持DoH。

#### 4) DNS-over-QUIC

它提供了DoT类似的安全特性，同时又有DNS-over-UDP的性能，目标是为达到最低延迟并防止如TCP线头阻塞 (head-of-line blocking) 的问题。但目前也并没有软件实现。

#### 5) DNSCrypt

DNSCrypt并非继续标准的TLS，而是X25519-XSalsa20Poly1305。并且使用DNSCrypt的client端需要安装额外软件 (e.g., DNSCrypt-proxy)，而server端需要提供专门的证书。

### 3 Server: 提供DoE服务

由于只有DoT和DoH被广泛支持并且已由IETF标准化，因此本文主要针对这两种方法进行研究。

#### 方法

##### 1) 发现公开的DoT解析服务器

由于DoT使用853端口，因此支持DoT的解析服务器必须能够接受发往这个端口的TCP连接请求。作者使用ZMap对整个IPv4地址的853端口进行了扫描，接着向这些IP发送DoT查询请求。从3个来自中国以及US的IP地址，每隔10天扫描一次，每次扫描正好需要24小时。

##### 2) 发现公开的DoH解析服务器

由于大多数DoH服务域名基本上都是二级域名的子域名 (e.g., dns.example.com)，因此无法通过查找公开zone文件来发现DoH服务域名。但由于DoH RFC定义解析服务器所使用的URL模板路径为`/dns-query`或`/resolve`，因此通过扫描URL数据集中指定的路径来寻找DoH解析服务。

![](/images/2019-12-10/4.PNG)

#### 发现

##### 1) 有1.5k的公开DoT解析服务来自大型提供商，但依旧有少数小型厂商提供了DoT服务，它们没有被包含在公开解析服务器列表里。相对地，DoH解析服务器数目少很多。

![](/images/2019-12-10/5.PNG)

对于DoH而言，作者经过扫描在他们的数据库中发现了61个保护了常用DoH路径的有效URL，除了常见的15个提供商外，还发现了新的两个DoH解析服务 (i.e., dns.adguard.com, dns.233py.com)。

##### 2) 25%的DoT提供商存在无效的SSL证书，相对地，所有的DoH服务器都良好地管理了证书。

![](/images/2019-12-10/6.PNG)

在最近一次扫描中，在总共122个解析服务器 (对应62个提供商) 使用了无效证书，其中27个已过期，67个属于自签证书，28个提供了无效证的书链。使用无效证书会导致DNS client无法验证server的身份，这是不安全的。

### 4 Client: 使用DoE服务

#### 方法

为了探究DoE服务的可用性，作者设置了许多用户端vantage points，这些点能够向公开DoE服务地址发送加密的DNS查询请求。

![](/images/2019-12-10/7.PNG)

作者使用ProxyRack，它提供了来自全球150个国家的600,000个终端SOCKS代理，因此除了可以对DoE获得一个全球性的认识，还可以探知网络审核国家对DNS加密数据的截获情况。

![](/images/2019-12-10/8.PNG)

考虑到测试的效率，作者只对三个公开DNS解析服务进行了测试，分别为: Cloudflare, Google, Quad9。同时，研究者还自己搭建了一个服务，支持明文DNS，DoT以及DoH解析。

![](/images/2019-12-10/9.PNG)

由于使用了proxy带来的延迟，研究者只能获得TN和TR的时间，但由于相对时间消耗是一致的，因此可以 通过控制查询的方式 (DNS/TCP或DoE) 来确认延迟情况。     

![](/images/2019-12-10/10.PNG)

#### 结果

##### 1) 可用性：全球99%的用户可以正常访问DoE服务，相对传统DNS而言，DoE被in-path设备影响的可能性更小。

DoE无法访问的可能原因分为如下：

+ IP地址冲突：原因可能是IP已经被in-path设备占用/IP被blackholed/或IP地址已被设为内部使用。比如说Cloudflare的1.1.1.1会被Cisco以及AT&T的设备占用，因此导致访问问题。
+ 互联网审查：可能对IP地址进行封锁/域名欺骗/连接重置，导致DNS响应被操纵。
+ TLS拦截：中间件以及杀毒软件可能会对TLS连接进行拦截并检查，导致DNS加密解除。

举例而言，16%的client不能获得Cloudflare的明文DNS回复，而DoT的失败率只有1.1%。                                                     

![](/images/2019-12-10/11.PNG)

![](/images/2019-12-10/12.PNG)

##### 2) 性能：DoE的时间延迟在不同国家之间是不同的

![](/images/2019-12-10/13.PNG)

DoE带来的延迟在70ms~500ms之间，而在有些国家 (如印度)，DoH的延迟比明文DNS要小。作者认为原因是DoH拥有由HTTP带来的更好的服务一致性，如拥塞控制与丢包重传。

### 5 使用: DoE流量

#### 方法

作者使用来自中国主路由收集的NetFlow数据集 (18个月)，从中收集了TCP端口853的记录，通过将目的地址与公开DoT解析器的IP配对以确定是DoT流量。对于DoH流量而言，由于在URI模板中查询的DoH服务域名要先被解析，因此作者通过两个DNS数据库: DNSDB以及360 Passive DNS来确定这些域名被查询的次数与世界变化。

#### 结果

##### DoE流量目前还是属于很小一部分

![](/images/2019-12-10/14.PNG)

![](/images/2019-12-10/15.PNG)
