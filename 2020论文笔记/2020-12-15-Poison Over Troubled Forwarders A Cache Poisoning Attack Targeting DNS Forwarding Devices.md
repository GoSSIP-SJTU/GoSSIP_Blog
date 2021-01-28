# Poison Over Troubled Forwarders: A Cache Poisoning Attack Targeting DNS Forwarding Devices 

> 作者：Xiaofeng Zheng, Chaoyi Lu, Jian Peng, Qiushi Yang, Dongjie Zhou, Baojun Liu, Keyu Man, Shuang Hao, Haixin Duan and Zhiyun Qian
> 单位：Tsinghua University; Qi An Xin Technology Research Institute; State Key Laboratory of Mathematical Engineering and Advanced Computing; University of California, Riverside; University of Texas at Dallas
> 会议：the Proceedings of the 29th USENIX Security Symposium
> 原文：https://www.usenix.org/system/files/sec20-zheng.pdf

[ToC]

## Abstract

在当代的 DNS 基础架构中，DNS 转发器在 DNS 客户端与 DNS 递归解释器中扮演着非常重要的角色。DNS 转发器一边相对于 DNS 客户端作为 DNS 服务器，一边转发客户端的 DNS 请求到其他服务器上。由于这种优势和许多应用场景，DNS 转发器被广泛部署。但是，作者的研究显示，DNS 转发器在 DNS 基础架构中更易受攻击。

在本文中，作者介绍了一种针对 DNS 转发器的缓存投毒攻击。通过这种攻击，恶意攻击者可以通过攻击者可控的域名来对特定域名注入恶意的记录，并绕过广泛部署的缓存投毒防御措施。通过在常用的家庭路由器和 DNS 转发器上测试，作者发现即便是很大的厂商（例如 TP-Link，Linksys，dnsmasq 和 MS DNS）的产品也出现了严重的安全漏洞。作者进一步在全球范围内进行了评估，估计出使用了存在漏洞的 DNS 转发器的移动客户端。

作者最后像多个厂商报告了这些问题，并收到了较好的反馈。作者的工作同时也证明了 DNS 转发器是 DNS 基础架构中的软肋，并呼吁社区和厂商遵循 DNS 转发器实现的标准。

<!-- more -->

## Introduction

DNS 服务是当今互联网基础建设的基石。与教科书上描写的 DNS 原理不同的是，现在 DNS 生态在当今互联网发展中变得十分复杂。对于 DNS 转发器来说，它处在 DNS 客户端与 DNS 服务器之间的位置。通常来说，DNS 转发器可以充当本地的 DNS 服务器用来提升解析速度，但是同时也带来了很多安全问题。

正是由于 DNS 转发器的便捷性，其被广泛部署在 DNS 基础建设中，例如极大一部分的家用路由器（例如 TP-Link，Linksys 和 D-Link 等）与众多 DNS 软件（例如 dnsmasq，BIND 等）。虽然 DNS 转发器的流行度非常高，但是对其进行的安全研究却很少。先前研究呼吁厂商和社区对 DNS 软件实现中增加针对缓存投毒的防御，例如源端口随机化、0x20 编码与 DNSSEC。

在本文中，作者通过提出一种缓存投毒攻击进一步证明了 DNS 转发器在 DNS 生态中存在的漏洞。利用作者所提出的方法，恶意攻击者可以利用其控制的域名来对指定域名的记录缓存进行投毒，并绕过了先前提出的众多针对缓存投毒的防御措施。作者在基于这种漏洞原理在众多硬件和软件上进行了测试，并发现当前绝大多数的硬件和软件都存在该问题。

**本文主要的贡献：**

1. 全新的攻击方式
2. 全新的 DNS 生态中的发现

## Prior DNS Cache Poisoning Attacks Targeting Recursive Resolvers

介绍先前针对 DNS 缓存投毒的相关研究

### DNS 报文伪造

攻击者直接伪造服务器响应报文，进行 DNS 抢答。

局限：源端口随机化、DNS 事务 ID 随机化、0x20 编码

### Defragmentation Attacks （分片合并攻击）

![](/images/2020-12-15/upload_5eaae31528294f05fc5529cad7fe3c28.png)

攻击者先发送第二个包分片，然后发起 DNS 查询。在权威 DNS 服务器返回时有两种攻击手段：

1. 攻击链路：降低 PMTU（路径 MTU）
2. 攻击特定域名：部署了 DNSSEC 的域名，因为其 DNS 包相对较大，可能超出以太网 MTU

局限：

1. PMTU 很难降低，难度很大，现实场景中不容易实际攻击成功
2. 仅能攻击部署了 DNSSEC 的域名

## DNS Forwarder

![](/images/2020-12-15/upload_ec31bc07924c470f82ef4b02008349fd.png)

传统 DNS 解析过程中，DNS 客户端发送解析请求给 DNS 递归解析器，然后由 DNS 递归解析器向域名权威服务器查询，并将查询数据返回给 DNS 客户端。然而，当今 DNS 生态要复杂的多，并会涉及到许多层服务，例如现在使用的非常广泛的 DNS 转发器。

DNS 转发器处于 DNS 客户端与递归服务器之间，它在接受到客户端解析请求后将查询转发给递归服务器，并将结果缓存起来。这样，DNS 转发器可以很大地提升客户端的解析速度，同时也可以用做上游递归解析器的负载均衡器、增加域名解析管控等。

1. 递归解释器会将 CNAME 链组装为一个 DNS 响应包，返回给 DNS 转发器
2. DNS 转发器通常不会发起递归查询，而依赖上游的响应

## Defragmentation Attacks Targeting DNS Forwarders

### 攻击模型

攻击者与用户处在同一局域网中，且局域网中使用了存在漏洞的 DNS 转发器。攻击者拥有完全可控的域名与其权威服务器。

![](/images/2020-12-15/upload_6e350cfc71a1c86cbbcbc6439e3e92f9.png)

### 强制 DNS 响应分片

**问题：**

1. 降低 PMTU 对 DNS 转发器效果甚微
2. DNSSEC 部署很少，且 DNS 递归解析器一般不会向 DNS 转发器转发 DNSSEC 请求

**解决方法：**

重点在于攻击者可控域名构造巨大的 CNAME 链，通过递归解析器返回给 DNS 转发器，并在返回时因为巨大的 IP 报文超过以太网 MTU 1500 字节限制被分片。

![](/images/2020-12-15/upload_b6b8596c1f2f780b3807a4492307868f.png)

由于攻击者预先发送第二个分片，DNS 转发器会将其缓存。在接收到递归解释器的 DNS 报文后将其第一个分片与之前缓存的第二个恶意分片组装，达到缓存投毒的目的。

### 构造伪造分片

![](/images/2020-12-15/upload_5a79f66609411e0d96486f699faa480c.png)

由于以太网这种分片中第二个分片不会包含 DNS 头，因此伪造难度大大降低，仅需要伪造 IPID 头和 UDP 头的 checksum 即可。

#### IPID 预测

IPID 是 IP 头里面 16 比特的一个字段，用来决定分片属于哪个报文。攻击者用于伪造的第二个包的 IPID 应当与 DNS 递归解析器返回的第一个分片的 IPID 一致，因此攻击者需要预测 DNS 递归解析器返回的报文的 IPID。

IPID 分配主要有三种：

1. 全局 IPID 计数器
2. 基于哈希的 IPID 计数器
3. 随机 IPID 分配

1. 第一种分配方法是完全可以预测的
2. 第二种基于哈希的 IPID 计数器算法首先使用一个哈希函数将一个发出的数据包映射到 IPID 计数器数组中的一个，然后将所选的计数器随机增加，增加的数量从 1 和自上次传输使用同一计数器的数据包以来的系统 tick（通常为毫秒）之间的统一区间中选择。如果两次请求非常接近，IPID 的递增就非常好预测。而事实上，因为分片组装可以缓存 64 个分片，攻击者可以在一个范围中猜测 IPID 的值。由于这种基于哈希的 IPID 计数器基于源 IP 和目的 IP，而又因为局域网中的请求都被 NAT 到同一公网地址，因此可以保证这些包使用了同一个哈希的 IPID 计数器
3. 第三种随机 IPID 则完全无法预测

![](/images/2020-12-15/upload_2b83fd11c40d4f58d09c6d2212ca8725.png)

#### 其他 IP 报文头字段

1. 分片偏移：攻击者完全可控
2. 源 IP 地址：提前发送请求获取响应的源 IP，再进行报文伪造
3. 第一个分片的 UDP 头 checksum：攻击者完全可控

### 攻击条件

1. EDNS(0) 支持
2. DNS 响应不会被缩减
3. DNS 响应不会被再校验
4. 基于记录的缓存（非整个 DNS 响应包）

## Vulnerable DNS Forwarder Software

### 家用路由器

![](/images/2020-12-15/upload_6800d5486cdd4b8a680df0d63e8f8723.png)

### DNS 软件

![](/images/2020-12-15/upload_3797397af616d88b9a7ef6d0b43aef1d.png)

## Client Population: A Nationwide Measurement Study

### 方法

![](/images/2020-12-15/upload_0ee588c4c4f5d36beec581eac6948f4e.png)

利用用户手机安装的 APP 对其家庭路由器进行测试，不过不实施实际攻击，利用 nonce 进行判断

![](/images/2020-12-15/upload_d712c354df7aa434b8881f28e26e9f62.png)

（隐私问题？）

### 分析

![](/images/2020-12-15/upload_699025f3aa92b84a512e7a273bf70360.png)

![](/images/2020-12-15/upload_6e1927968a2edf5689d6547f5ccd1f01.png)

数据显示有 6.6% 的 DNS 转发器存在本文所提出的攻击。不同于先前的研究，本文的攻击方式不受任何域名的限制。

在进一步分析时，作者发现漏洞没有触发是因为 DNS 转发器没有适配 EDNS(0) 支持（40.8%）和正确处理过大的响应包（28.3%）。由于现在 DNS feature 逐渐增多，这种漏洞会影响到更多的用户。

## Reflection on DNS Forwarders

![](/images/2020-12-15/upload_52e1673c4e3100a0d5b301eebd2bdfb0.png)

难怪这么难实现（RFC 太多了

主要问题：缺乏 DNS 转发器实现的 guideline （RFC 5625）

## Attack Model Extension and Mitigation

### Extending the Attack Model

![](/images/2020-12-15/upload_367fa15c140cc54e4e6230c160515250.png)

攻击面扩展，作者可以对公网上的 DNS 转发器进行缓存投毒，其造成的影响更大

### Mitigation

1. 响应验证：重新查询每一条记录
2. 以响应缓存 DNS（而不是以记录缓存）
3. DNS 记录以 0x20 编码：编码所有的 DNS 记录
4. 随机 IPID

## Conclusion

主要是全新的 DNS 攻击手段，针对部署非常广泛的 DNS 转发器。作者在全国范围内的测试发现问题较为严重。


