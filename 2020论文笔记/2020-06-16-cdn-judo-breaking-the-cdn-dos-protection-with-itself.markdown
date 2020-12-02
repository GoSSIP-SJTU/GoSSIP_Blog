---
layout: post
title: "CDN Judo: Breaking the CDN DoS Protection with Itself"
date: 2020-06-16 15:17:11 +0800
comments: true
categories: 
---

> 作者：Run Guo, Weizhong Li, Baojun Liu, Shuang Hao, Jia Zhang, Haixin Duan, Kaiwen Shen, Jianjun Chen, Ying Liu
> 
> 单位：Tsinghua University, CERNET, University of Texas at Dallas, ICSI (International Computer Science Institute, BNRist (Beijing National Research Center for Information Science and Technology
> 
> 会议：The Network and Distributed System Security Symposium (NDSS) 2020
> 
> 原文：https://www.ndss-symposium.org/wp-content/uploads/2020/02/24411-paper.pdf

## Abstract

内容分发网络 `(Content Delivery Network)`，即 CDN，其依赖于全球分布式网络，可以给网站带来更高的访问性能和全球高可用的网络基建。因为 CDN 通常为极其重要的服务或者商用网站来提供服务，攻击者可能更感兴趣于攻击甚至拿下这些高价值的网站从而使得网站运营者承受大量的损失。因为 CDN 服务商通常有巨大的分布式网络带宽资源，可以吸收大量的分布式攻击流量，人们普遍认为 CDN 可以给 DoS `(Denial of Service)` 攻击提供可靠的保护。

但是，作者发现有部分 CDN 服务商在转发机制上存在实现缺陷或协议缺陷，使得攻击者可以攻破 CDN 的保护机制。通过向 CDN 发送精心构造好的但是合法的数据包，攻击者可以对在 CDN 保护下的源站发起有效的 DoS 攻击。

具体来说有以下三种攻击途径：

1. 滥用 CDN 的 `HTTP/2` 协议转换
2. 滥用 CDN 的 `pre-POST` 发包机制
3. 滥用 CDN 的入站 IP 和出站 IP 数量差距与出站 IP 的切换频率

通过上述 1，2 两点，攻击者可以塞满 CDN 到源站的带宽，占满源站服务器的并发连接数量，从而造成源站 DoS。通过上述第 3 点，攻击者可以有效的降低网站的全球可用性。

这篇论文在常用的，或者说占市场份额较高的 6 个 CDN 服务商进行了上述三种测试，并在实际的使用场景下发起上述三种攻击，评估了攻击造成的实际影响。由于这些威胁是由 CDN 厂商在可用性和安全性之间的取舍造成的缺陷，作者讨论了可能的缓解措施并报告给了上述 CDN 服务商，并收到了积极的反馈。

<!-- more -->

## Introduction

通过全球性部署的代理服务器和巨大带宽资源的网络基建，CDN 可以给仅有一个地域性 IP 的网站提供全球可用，高性能的服务。由于流量分流和全球可用的特性，CDN 已经成为互联网生态中不可或缺的一部分。正是由于其服务的特性，CDN 可以防御 DoS 攻击，从而使得 CDN 服务在互联网生态中极大地扩展。在 Alexa 排行榜上，前 1k 名中 50% 使用了 CDN，前 10k 名中 35% 使用了 CDN。

在这篇文章中，作者测试了 6 个 CDN 服务商。通过向 CDN 发送精心构造的但是合法的数据包，作者证明了 CDN 本身可以用来滥用并向源站发起 DoS 攻击，攻破 CDN 的 DoS 保护作用。

1. HTTP/2 带宽放大攻击

    作者发现 CDN 仅仅在客户端到 CDN 服务端的连接中使用 `HTTP/2`，也就是意味着在 CDN 回源时需要将 `HTTP/2` 转换为 `HTTP/1.1` 协议，从而造成了一种带宽放大攻击。作者同时还分析了 `HTTP/2` 引入的 `HPACK` 压缩方法对带宽放大攻击的影响。

2.  pre-POST 缓慢 HTTP 攻击

    作者发现在上述 6 个 CDN 服务商中，有 3 个会在接收到 POST 请求时没有等待 POST 包的 Body 接收完毕再发送，而是立即转发给源服务器。这种机制可以被滥用从而耗尽源服务器的并发连接数量，使得服务器无法处理正常用户的请求。更糟糕的是，这些 CDN 的 `HTTP/1.1` 和 `HTTP/2` 的 POST 包转发机制都会受到这种攻击的影响。

3. 全球可用降级攻击

    作者使用互联网际扫描器对 CDN 服务商的入站 IP 进行大规模扫描，并对 CDN 服务商的出站 IP 进行收集。通过对收集到的入站和出站 IP 数量进行分析，作者发现出站 IP 数量要远少于 入站 IP 数量，并且出站 IP 的切换率很低。因此，攻击者只需要切断一个或部分出站 IP 到源站之间的连接即可造成网站全球性不可用。拿 MaxCDN 来说，MaxCDN 只有一个出站 IP，也就是说只要攻击者切断这一个出站 IP 到源站之间的连接即可造成 90% 以上的全球性不可用。

总结起来说，作者专注于如何攻破 CDN 的保护功能，这对很多网站来说是极有价值的。DoS 可以对网站甚至互联网造成非常严重的损害，不仅仅在于经济损失，还有客户们对网站的信任。作者所做的工作意在帮助 CDN 服务商提高他们的安全意识。同时，作者也有义务公开他们的发现并报告给受到影响的 CDN 服务商，此举也收到了 CDN 服务商们对其工作的积极的反馈。

## Background & Threat Model

### Background

#### CDN

CDN 使用全球性部署的代理服务器和极大带宽资源的网络基建为普通网站提供全球性可用的服务。

在收到了客户端的请求后，CDN 会检查请求的 HTTP 头，尤其是 `HOST` 和 `URI`。如果部分资源已经被缓存，CDN 会将资源直接发送给客户端，否则 CDN 会将请求转发给源站获得资源，并返回给客户端。CDN 起到了一种中间人的作用，在客户端和源站之间。

![](https://docs.evi0s.com/uploads/upload_57e6c2669834ee317dc5ce516a8acb74.png)

由上述过程可以看出，CDN 需要对客户端协议进行转换。同时为了提高客户端速度，CDN 需要尽可能快的把客户端请求转发给源站，并需要利用好缓存机制。

#### 请求路由机制

请求路由机制对 CDN 来说至关重要，因为 CDN 依靠这种机制来为客户分配最近的节点。但是这种机制可以被绕过，只需要直接访问 CDN 入站 IP，并指定好 `HOST` 头即可。

#### HTTP/2

由于 HTTP 协议中的 `Header` 存在大部分重复的字段，`HTTP/2` 使用 Header 压缩和单连接复用来提升网络性能。

![](https://docs.evi0s.com/uploads/upload_cf08374dd33cf503bbe29aa4f91ac22f.svg)

![](https://docs.evi0s.com/uploads/upload_98dc7dd74dee7272d08b5ac3503183cc.svg)

#### CDN 简单对比

作者选用了 CDN 市场上占有率比较高的 6 个 CDN 服务商：

市场关键服务商：

* CloudFront
* Cloudflare
* Fastly

提供个人免费试用：

* CDNSun
* KeyCDN
* MaxCDN

除了 CloudFront 需要信用卡验证之外，其他仅仅使用邮箱即可注册使用。这使得攻击者的攻击成本降低，也降低了暴露个人隐私信息的风险。

### Treat Model

![](https://docs.evi0s.com/uploads/upload_1c2654cbac671b0d15ed9de31583a8df.png)

通过向 CDN 发送合法的数据包发起对源站的攻击。

## HTTP/2 带宽放大攻击

### 攻击面分析

CDN 通常支持 `HTTP/2`，但是仅仅是客户端和 CDN 之间的连接使用 `HTTP/2`。在回源时，即使源服务器支持 `HTTP/2`，CDN 仍然会将客户端请求的 `HTTP/2` 转为 `HTTP/1.1` 再访问源站。这种行为可能会导致安全隐患，而更糟糕的是，调查的 6 个 CDN 服务商除了 Fastly，都是默认开启 `HTTP/2` 的选项，甚至部分无法关闭。

![](https://docs.evi0s.com/uploads/upload_cba56052dd319aebac40126ffcdae939.png)

![](https://docs.evi0s.com/uploads/upload_f989510542051dad62fd2c3dcf9543bc.png)

由之前介绍的 `HTTP/2` 可知，`HTTP/2` 不会发送重复出现的 HTTP Header，并使用 `HPACK` 来压缩 Header。这样会导致，客户端在发送同样 Header 的请求时，相同内容被忽略，不同的内容被压缩。但是在经过 CDN 的 `HTTP/2` 转换为 `HTTP/1.1` 时，Header 被解压，同样的内容需要多次发送给源站。这样就导致了小带宽流量被 CDN 放大后发送给源站，从而导致源站 DoS。

![](https://docs.evi0s.com/uploads/upload_f70d8c34741ff9fee672d225b2276e46.png)

由于 `HTTP/2` 的特性，攻击者可以在一个 `HTTP/2` 的 TCP 连接中并行发送多个 stream。只有第一个 stream 中包含大量数据的 cookie 头（或 user-agent，referrer 头等），而其他仅需要发送不同的 path，即可使 CDN 转发大量流量到源站。

根据 RFC 标准，合法的 `HTTP/2` 协议中对最大输入有一定的限制，但是并行 stream 数量则因 CDN 服务商而异。

![](https://docs.evi0s.com/uploads/upload_d7f767d5dd75aca419d7f4212e705e55.png)

因此，攻击者可以试图以 CDN 服务商的最高并行 stream 数量来并行发送少量请求，并通过 CDN 的协议转换放大作用达到最高的放大比例。这些包将被 CDN 转发给源站最终使其 DoS。

### 现实攻击场景

作者构造了这样的一个 `HTTP/2` 数据包

```
:path: /?<random_string> (or /)
:scheme: https
:authority: victim.com
:method: GET
cookie: A=X...X (a large-sized string)
cookie: B=X...X (a large-sized string)
```

为了达到最高放大率，作者使用了两个 `cookie` 字段去填满 `HTTP/2` 动态表，加起来一共是 3072 个字节，然后使用 `curl` 逐渐增大并行 stream 数量去发送这些数据包。同时，在源站使用 `tcpdump` 去记录流量大小。最终获得了实验结果：

![](https://docs.evi0s.com/uploads/upload_4f5a8bf6c8bb3f202955e4db4441992f.png)

![](https://docs.evi0s.com/uploads/upload_343b9473f370bed76ae88117181e00ce.png)

从上面的结果来看，这个放大攻击（80 - 130 倍）效果还是很可观的（可以对比 UDP 的放大攻击）。

作者在得出实验结果后，分析得到两个影响到放大倍率的因素

#### 哈夫曼编码

在 `HPACK` 压缩方法中，哈夫曼编码会进一步对 Header 压缩。其中最短是 5 bit，最长 8 bit，也就是说最高压缩比是 8:5 即要小 37.5%。这样，攻击者发送的包经过 CDN 放大后，Header 的大小还可以放大最高 8/5 倍即要大 160%。

因此，在作者实验中，`cookie` 使用 `0, 1, 2, a, c, e, i, o, s, t` 这些字符（即哈夫曼编码后为 5 bit）可以达到最大的放大率。

&nbsp;

#### :path 选择

在 `:path` 选择上，可以直接选择发送 `:path /` 或者使用一个随机字符串的 query `:path /?random_string`。大小差异如下表所示：

![](https://docs.evi0s.com/uploads/upload_8eeada47b039e47189164e8eef4d5a42.png)

直接使用 `:path /` 来攻击源站，会给其他 Header 例如 `cookie` 留下更大空间来放大发送给源站，但是可能会存在别的问题：

* **CDN WAF：** CDN 通常会有一定的 WAF 规则，而这种请求很容易被 WAF 探测并拦截
* **CDN Cache：** CDN 一般会将这种重复提交且相同 path 的请求给缓存，不发送给源站

设攻击者到 CDN 的攻击带宽为 $FB$（Front-End 即 attacker-CDN），CDN 到源站放大后带宽为 $BB$（Back-End 即 CDN-origin）。那么可以得到放大率为：

$$
\frac{BB}{FB}
$$

如果攻击者要多发送一个 `random_string` 的话，就存在一个多余的带宽开销：假设有 $n$ 个 `HTTP/2` 的并行 stream，每个 stream 就需要多发送 $8 - 1 = 7$ 个字节的数据（`:path` 索引是 1 字节，一共 8 字节），即攻击者一共要多发送 $7n$ 个字节。在经过 CDN 转换为 `HTTP/1.1` 协议之后，CDN 需要多发送 $7 - 1 = 6$ 个字节。这样就可以得到一个不等式：

$$
 \frac{BB}{FB} > \frac{BB + 6n}{FB + 7n} 
$$

可以假设 $FB = 5000$，$BB=600000$，可以看出放大率会随着 `HTTP/2` 并行 stream 的增大而减小：

$$
\frac{BB}{FB} = \frac{600000}{5000} > \frac{600000 + 128 \times 6}{5000 + 128 \times 7} = 101.9 > \frac{600000 + 256 \times 6}{5000 + 256 \times 7} = 88.6 
$$

作者也对其做了实验进行了验证，实验结果如下表

![](https://docs.evi0s.com/uploads/upload_d77d633658e16b945b092fe4e47c4691.png)

### 总结

这种 DoS 放大攻击效果可观，尤其是再结合 CDN 自己的服务节点，可以利用 CDN 本身升级为一次全球性的 DDoS 攻击，可以极大影响源站的性能。同时再结合 CDN 本身默认打开的 `HTTP/2` 支持，这种攻击威胁巨大，且后果严重。

## pre-POST 缓慢 HTTP 攻击

### 攻击面分析

pre-POST 攻击已经不是什么新鲜的攻击方法了。相比较于传统的洪泛攻击，这种攻击方式更加隐蔽且更加高效。

攻击者发送第一个 POST 包，其中包含了 `Content-Length` 大于第一个包的 body 长度的包，这时，服务器将会等待攻击者发送完毕。这样，攻击者可以每隔一秒发送一个字节，保持客户端与服务器之间的连接。由此，攻击者可以不断地创建这种连接，最终耗尽后端服务器的最大并发连接数（例如 Apache 服务器默认最高连接数为 1000）。

通常来说，CDN 会解耦客户端到 CDN 之间的连接和 CDN 到源站的连接，这样可以吸收掉洪泛流量，从而防御 DoS 攻击。但是在作者的研究中，作者发现不同 CDN 会对 pre-POST 包有不同的处理方法：有的 CDN 在收到客户端 POST 请求后立即转发流量到源站，有些则会等待客户端将完整 POST 包发送完毕后再转发给源站。

### 现实攻击场景

作者设法构造了这样一个数据包：

```
POST /login.php?<random_string> HTTP/1.1
Host: www.victim.com
Content-Length: 300

0101..... (300 bytes, 1 byte sent per second)
```

同时在源站用 `tcpdump` 来检查 CDN 转发包到源站的时间

最终得到实验结果如下：

![](https://docs.evi0s.com/uploads/upload_5c57e2a0b8a454eee338772358f8f6bd.png)

很明显可以看到，CloudFront，Fastly 和 MaxCDN 都在接收到客户端的 POST 请求之后立即转发给源站。不过 MaxCDN 仅仅会保持 15 秒的连接，就断开了与两者之间的连接。

作者给源站部署了一个 Apache 服务器，设定最大连接数为 1000 个即默认值，并测试并发连接数量，同时模拟普通用户访问网站并测定延迟大小，结果如下：

![](https://docs.evi0s.com/uploads/upload_56bc81719a5f05f8600a7a76974b6b0a.png)

![](https://docs.evi0s.com/uploads/upload_0862f3caa34a9461e7c2d51973218c2a.png)

可以看到，普通用户因为 pre-POST 攻击，他们对网站访问的延迟大大增加。CloudFront 会在 90 秒后返回 504 Gateway Time-out，Fastly 会在 15 秒后返回 503 Service Unavailable。而 MaxCDN 因为有 QoS，并且有 15 秒的超时，因此普通用户访问的响应速度会低于 15 秒，相对来说要好一点。

由于上一章作者阐述了 HTTP/2 相关的攻击，在这一章作者同样使用 HTTP/2 发起 pre-POST 缓慢 HTTP 攻击。而这一次 Fastly 的结果稍有变化，因为 Fastly 仅仅维护 10 个并行 stream，而其他的 stream 则被丢进等待队列中，会等待客户端 POST 包发送完毕。具体结果如下：

![](https://docs.evi0s.com/uploads/upload_447cbdfe8096ca98195cd7221a7d7375.png)

### 总结

在现实环境中，6 个 CDN 服务商中有 3 个有这种问题出现，这将成为一个较大的安全隐患。对于这 3 个 CDN 服务商，攻击者可以直接使用 pre-POST 攻击源站，攻破 CDN 的保护功能。

## 全球可用降级攻击

### CDN 入站与出站 IP 分布

作者首先 WHOIS 信息来收集，在收集了部分信息后，再用 Censys 这个互联网际扫描工具发送不正确的 IP 格式的 HOST 头来进行大规模扫描。由于这些 CDN 的入站 IP 的响应有明显的特征，因此相对容易收集：

![](https://docs.evi0s.com/uploads/upload_712b1bff44e807316eeacd0c47b126ea.png)

然后再用正确的网站域名填充 `HOST` 再访问这些 IP，如果能正常访问即为正确的 CDN 入站 IP。

对于出站 IP，作者使用正确地 `HOST` 头访问之前收集到的入站 IP，并在 24 小时内不停发送这些包，并在源站收集收到请求的 IP 地址。

![](https://docs.evi0s.com/uploads/upload_b6b47ed02c71ad24bba37a55e7c9046e.png)

从收集到的结果可以看到，CDN 通常使用大量的入站 IP 来接收客户端的请求，但是用来回源的出站 IP 要远小于入站 IP 数量。

于此同时，作者还对 24 小时内出站 IP 切换的频率进行了统计，Y 轴是在所有出站 IP 中分配到该 IP 在 24 小时内所占比例：

![](https://docs.evi0s.com/uploads/upload_a5cf304382509e6c5eb71f3053730c41.png)

可以看到，MaxCDN 几乎只用一个出站 IP 地址去请求源站，而其他的 CDN 则较为随机地分配出站 IP 去请求源站。还有一个地域问题，即 CDN 会尽可能地分配与源站地域相同或者相近的出站 IP。

L. Jin 等其他研究者也对这些问题进行了研究，他们发现入站与出站 IP 不相同的原因需要归结到 CDN 的二层架构上，即面向客户数量要远远多于源站 IP 数量。这种架构可以增加缓存命中率，并减小源站的负载。但是显然这种架构存在一定的安全风险，使得攻击者切断 CDN 到源站的连接要方便的多。

### 攻击面分析

CDN 通过其全球分布的入站节点可以为源站提供全球可用性，但是由上节所述，低切换率的出站 IP 可以给攻击者带来很大的便利来切断 CDN 到源站的连接，从而造成全球可用性降级。但是有一个问题在于，源站 IP 一般很难获知，那么只能通过社工或者信息泄露等方法去获取源站 IP。甚至，攻击者可以试图去通过攻击路由器或者防火墙去切断 CDN 到源站路径上的链路来达到目的。

![](https://docs.evi0s.com/uploads/upload_032be0d0c54dc7e1981377b437609662.png)

通过上图可以看到，由于低切换率的出站 IP，攻击者仅需要攻击少数几个 CDN 出站 IP 到源站之间的连接，即可切断全球 CDN 节点对源站的访问。

其实说起来容易，作者也并不认为这种攻击方法是实用的。因为这需要攻击者极具侵略性，并且至少掌握多个 CDN 出站 IP 到源站沿路节点的路由器或者防火墙的控制权。但是这种攻击还可以用另外一种前几年被提出的，叫 [Crossfire](https://www.ieee-security.org/TC/SP2013/papers/4977a127.pdf) 的链路层攻击来实现。

![](https://docs.evi0s.com/uploads/upload_ca61f17aa3adbe67ff81cfe2fa08faad.png)

&nbsp;

&nbsp;

### 现实攻击场景

这里，作者找到了一种讨巧的办法，即天然的防火墙 -- GFW，来完成这个方法的现实场景攻击测试。

![](https://docs.evi0s.com/uploads/upload_d7586002f4d715d90028222877f6fa47.png)

如果源站在 GFW 内，即 CDN 需要跨 GFW 访问源站，那么攻击者可以使用明文协议，例如 HTTP 协议 (?) 来发送一些敏感词。当 CDN 将攻击者发送的敏感词转发给源站时，由于 GFW 对明文内容的过滤，会立即给 CDN 发送 RST 来中断连接，并加入 IP 黑名单。这样攻击者就成功地将 CDN 出站 IP 到源站之间地连接给切断，使得全球其他正常用户也无法访问该网站。

同时，作者还实际测试了一下这个实验，下图可以看到，由于 MaxCDN 的出站 IP 数量过少，被 GFW ban 掉这些 IP 后，全球几乎所有节点都无法访问受攻击的网站。

![](https://docs.evi0s.com/uploads/upload_fd11214ff1de4243b4ae8ad92130c8bf.png)

### 总结

由于 CDN 服务商入站与出站 IP 数量相当不均衡，及出站 IP 切换速率过低，攻击者可以使用例如 Crossfire 或者滥用 GFW 来攻击 CDN 的出站 IP 到源站之间的连接，从而使得网站全球可用性降级。因此，这种威胁要比看起来要严重很多。

&nbsp;

&nbsp;

## 讨论

### 严重性分析

由于 CDN 通常被认为是用来保护源站的，因此很多源站的防火墙策略仅仅允许 CDN 发起对源站的请求。这也就意味着，源站一般是绝对信任 CDN 的。但是本文阐述的三种 DoS 攻击可以利用 CDN 本身来发起对源站的 DoS 攻击，或者使该站点 DoS，都会造成极其严重的后果。

同时，如果 CDN 服务商不对网站拥有者进行校验，恶意攻击者甚至可以配置其他源站通过 CDN，即：攻击者不仅可以通过 CDN 发起对已经配置好源站的网站进行攻击，甚至可以配置其他 IP 地址来利用 CDN 发起 DoS 攻击。

### 缓解措施

![](https://docs.evi0s.com/uploads/upload_1429b2e3d855ec761d20282a710f8855.png)

* **HTTP/2 攻击：** 关闭客户端访问 CDN 的 `HTTP/2` 选项，或者限制 CDN 回源频率
* **Pre-POST 攻击：** 限制 CDN 回源连接数，强制完整接收后再转发
* **全球可用性降级攻击：** 提高不可预知性的出站 IP 的切换速率

## 结论

在作者文章中测试的 6 个 CDN 中，每个 CDN 都或多或少有安全隐患。这种隐患可以使用来保护源站的 CDN 本身成为发起 DoS 攻击源站的工具。这些问题应当被 CDN 服务商重视起来，防止这些隐患在现实中被真正利用，造成损失。
