---
layout: post
title: "On Measuring RPKI Relying Parties"
date: 2020-11-10 15:38:48 +0800
comments: true
categories: 
---

> 作者&部门：John Kristoff, Chris Kanich (University of Illinois at Chicago USA,伊利诺伊大学芝加哥分校)
> Randy Bush (IIJ and Arrcus Japan and USA)
> George Michaelson (APNIC Australia)
> Amreesh Phokeer (AFRINIC Mauritius)
> Thomas C. Schmidt (HAW Hamburg Germany)
> Matthias Wählisch (Freie Universität Berlin Germany)
> 
> 会议：IMC 2020
> 
> 论文：[link](https://dl.acm.org/doi/pdf/10.1145/3419394.3423622)

在本篇论文中，作者对RPKI中PP和RP软件的部署进行了完整性和一致性的调查。

<!-- more -->

## RPKI

- BGP简述和其安全问题

    BGP(border gateway protocol)用于在不同的自治系统（AS）之间交换路由信息。举个例子，`129.21.0.0/16, AS_PATH: AS3549 AS3356 AS4385`，表示到达IP前缀为129.21.0.0所需经过的一串AS路径。

    BGP缺乏一个安全可信的路由认证机制，即BGP默认接受任何AS发起的路由通告，因此可能存在劫持攻击。BGP劫持指的是AS对外通告一个未获授权的前缀，所谓“未获授权”指的是前缀属于其他AS所有或者该段地址空间尚未分配，使得流量流向了攻击者设计的方向。

- RPKI

    RPKI(resource public key infrastructure)是一个提高BGP路由系统安全性的框架。在RPKI中，证书与AS编号和IP地址绑定在一起，以保证BGP不被劫持。

    - 五大区域互联网中心（Regional Internet Registries ，RIR）都是自己区域内的信任锚（等价于根证书），他们的私钥用于签发证书。
    - 发布节点（Publication Point，PP）服务器，用于将RPKI数据提供给依赖方(RPs)。
    - RPKI对象，包括了证书，ROA，支持结构（清单，CRLs）。
        - 证书，它将Internet资源编号，如Autonomous System Numbers (ASNs)和IP前缀和公钥绑定。
        - ROA（Route Oringin Authorization，ROA）允许证书持有者使用证书认证自己宣称的IP前缀。
        - 清单，它列出了其所在资料库发布点所收集的由权威结构发布的所有签名对象的信息。
        所有的PKI对象以文件的形式由发布点（publication point，PP）服务器散播。
    - RPKI仓库，它是一个分布式存储结构，是由很多数据库组成的分布式数据系统。
    - 依赖方（Relying Party，RP）是一个用来下载和验证RPKI的软件，RP使用rsync或RRDP进行数据检索。执行RPKI证书和签名对象验证的实体成为依赖方。依赖方访问RPKI资料库、获取RPKI证书和签名对象，直接指导边界路由器判断在互联网中广播的路由通告是否得到了合法授权。RP可以不是RPKI用户。

![](/images/2020-11-10/uploads/upload_0584de54a4c45724e5cf2cd7b539fd6d.png)

现在，几大路由中转商（AT&T，NTT，Telia），互联交换中心（AMS-IX），中小型ISP（Fiber Telecom）和内容提供商（Cloudflare）都会根据RPKI信息评估和拒绝无效路由。

## 2. Measurement Framework
作者研究的对象包括了：被控制的CA（亚太互联网信息中心APNIC，非洲互联网信息中心AFRINIC）和发布节点，被控制的依赖方，被控制的RPKI对象。

- 作者首先在每个区域互联网中心下建立了一个子节点，两个孙子节点，和三个发布节点。
- 作者部署了多个拓扑分布的RP，每个RP运行不同的软件，并从信任锚中递归获取数据。
- 作者建立了ROA Beacons，用于在合适的时间改变以上的设置，如发布BGP对象或撤销BGP对象，以检测延迟。

作者自2018年开始运行CA/PP和ROA Beacons，而自2019年12月开始控制RP运行。

## 3. Results

### 3.1 Completeness of RPKI View

作者首先回答了一个问题：RP是否从每个PP提取数据？作者通过对访问其控制的PP的RP的IP地址进行计数来衡量。

结果如图二所示，作者观察到了如下现象：
 - 可以发现随着时间的推移，收集到的IP数目不断增多。
 - 在2019年年末，使用rsync的数目有所减少，这是因为RRDP被激活，被现代RP软件所青睐。但是由于锚点配置中要求rsync链接的原因，rsync在RIR的数据集中RRDP的链接数并没有明显减少。
 - 可以发现两个RIR的发布节点收集到的IP地址和作者的发布节点收集到的IP地址并不相同，这表明并不是所有的RP都保存了完整的RPKI数据集。
 - 作者部署的发布节点收集到的RP的IP地址更少一些，经过和RIR的数据集进行对比，作者认为是RP没有对PP子节点进行询问而导致的。

![](/images/2020-11-10/uploads/upload_1980dca357022701525187dfee1d889a.png)


### 3.2 Type of Networks Hosting RPs

在本节中，作者关注了RP是被谁部署的，以及部署的数量。

RPKI规范建议了三种部署RP的模型，分别是：由提供商或第三方提供；在本地缓存系统中部署；在网络中部署一组分布式的缓存系统。在本次实验中，作者发现大多数运营商为整个网络部署了一个或者两个不同类型的RP。少数则部署了自己设计的RP。

如图3所示，RP的增长趋势主要是由以下三个部门驱动的：Cable/DSL/ISP，Content(Facebook，从2019年的没有部署RP上升到2020年初的部署了近70个RP。)， NSP(network service provider，如Linode和DigitalOcean)。
![](/images/2020-11-10/uploads/upload_0f33c61569b1279a71346197b035f71f.png)


为了了解网络服务的RP部署情况，作者绘制了2020年前4个月内的每个ASN的不同RP数目的累积分布函数图。作者通过对RP的IP地址进行统计，发现大多数RP都可以被认为是具有稳定IP地址的服务器系统。
![](/images/2020-11-10/uploads/upload_b019f34a024bb4cddf285afffb2c9d88.png)


在一般情况下，有的网络服务提供商会部署十多个RP。作者审查到有一个网络供应商拥有超过1000个RP，作者联系了该网络提供商，他们解释说他们使用了基于容器的模型部署了十几个RP，这些只是临时节点，这将导致随着网络的变化，地址可能在短时间内频繁更改。

作者还检测到了洋葱路下RP的实际IP地址被隐藏。

### 3.3 Timeliness

由于RP通过刷新来获得RPKI视图，因此刷新是否及时是衡量能否获得RPKI完整视图的重要因素。因此在本节，作者评估了RP的刷新频率。现在的RP软件默认将刷新间隔设置为一小时或更短。
![](/images/2020-11-10/uploads/upload_f342d25533eaeff33ef3de5ff6402941.png)

作者在2019年和2020年，对RP的刷新频率进行了统计，结果如图5和图6所示。
![](/images/2020-11-10/uploads/upload_470ba9b2bf406090a95bf706ec638e71.png)

作者观察到了如下现象：
- RRDP的刷新频率较为稳定，而rsync的刷新频率噪声较多。
- 在2019年，rsync是RP用于询问的主要方法，但由于询问时间可以被自定义，其刷新频率不可被预测。后期RRDP被采用后，很多RP选择了RP和RRDP的组合，因此对于使用rsync链接的RP，作者都可以观察到RRDP的指纹。


### 3.4 Filtering Experiment
本节作者讨论了PP和RP在容错性方面的表现。

对于PP，PP运营商一般会通过DNS轮询制度或负载均衡来降低错误的发生，同时RP也会在每个同步间隔内询问PP。因此PP的容错能力较高。

所以，作者在这里主要讨论了RP软件是否拥有处理瞬间连接失败的能力。如前，大多数RP同时提供RRDP(基于http的服务)和rsync(TCP端口873)的服务接口。5月8日，作者在子节点PP-a上阻止了RP对RRDP服务的访问，并在另一个子节点PP-b上阻止了rsync。5月13日，作者交换了对每个服务器上的协议阻塞。

如图7所示，许多RP不使用符合当前规范的实现。当RP发现RRDP失败时，并没有调整到rsync进行询问。

![](/images/2020-11-10/uploads/upload_beecc00666a983a05703fd031bf3ad9a.png)

## Conclusion
作者得到了已部署的RP中大部分未获得RPKI的完整视图。作者后面的工作：
- RP是否遵守证书的到期日期？
- RP是否正确验证所有的ROA？
- RP在验证过程中是否遵守规则？