---
layout: post
title: "Let’s Revoke: Scalable Global Certificate Revocation"
date: 2020-09-15 14:06:42 +0800
comments: true
categories: 
---

> 作者：Trevor Smith, Luke Dickinson, Kent Seamons
> 
> 组织：Brigham Young University
> 
> 会议：USENIX 2020
> 
> 原文：https://www.ndss-symposium.org/wp-content/uploads/2020/02/24084.pdf

## Abstract

当前的撤销策略存在许多问题，包括可伸缩性、私密性和新的基础设施需求。撤销往往被忽视,使客户容易受到中间人攻击。本文介绍了一种可伸缩的全局撤销策略Let 's Revoke，该策略解决了当前撤销检查的问题。与现有的撤销方案进行比较，所需的存储和网络带宽更少。作者通过模拟进一步证明，即使在大规模的撤销事件中，也可以线性扩展到100亿个证书。

<!-- more -->

## Related Work

### Pull-Based Revocation

常见的有两种，CRLs和OCSP。对于CRL，但是由于被撤销的证书太多、证书列表过大，很多浏览器都不再采用该方式来检查网站证书是否被撤销。对于OCSP，会话会存在延时，客户端对OCSP的会话请求会泄露客户的隐私。

在基于pull的机制中，一方面客户端一般会缓存撤销的记录以加快网页的加载，对于没有缓存撤销记录的情况，会存在中间人攻击。

### Push-Based Revocation

对于基于Push的策略，客户端定期下载撤销的信息。Google CRLset和Mozilla OneCRL都基于了这种策略。CRLset是由Google内部构建的列表，依据证书的撤销原因存储了被撤销的叶子证书，它的大小为250KB，包含了40,000张证书。但是CRLSets对防止证书伪造做出了贡献，但它不能保护大多数证书；由Mozilla设计的OneCRL有相似的功能，其存储的证书也不仅限于叶子证书。

对于其他降低带宽的方法，还包括了使用更高效的数据结构，如布隆过滤器。

### Network-Assisted Revocation

减少了客户端的努力，主要改变TLS环境来解决撤销问题。

比如把撤销的信息插入在CDN中，而CDN把撤销的信息作为TLS extension插入到通信中。比如使用Certificate Revocation Guard (CRG)，但是不适用于笔记本电脑和移动手机。比如OCSP Stapling，但容易受到假冒和中间人攻击，因为攻击者选择在与客户端握手时不包含OCSP Must-Stapling。比如OCSP-Must Staple机制解决了这种攻击，但是如果全面使用OCSP Must-Staple，则可能存在对OCSP的应答机制进行DOS攻击而使网络瘫痪的情况，因此Google不支持OCSP Must-Staple，有人提出使用CDN解决这一问题，但会存在CA不一致以及服务器实施的困难。比如，使用短期证书，以及频繁更新证书密钥，但是存在需要人工操作的问题。

## LET'S REVOKE SYSTEM DESIGN

Certificate Revocation Vectors (CRVs)，是简单的数据结构，使用一位表示证书的撤销状态(0-no, 1-yes)。

### A. Design Description

**Revocation Numbers**表示特定CRV中撤销位的索引。

**Certificate Revocation Vectors**Let's Revoke维护了一个CRV的数据库，每个CA都有CRV记录的过期日期。

举个例子，Example CA在2019年1月颁发了16张证书，在2020年1月即将过期。CA为这16张证书设置了0-15的Revocation Number。在2019难2月，RN编号为7的证书所有者提出证书撤销的申请，ECA撤销了该证书，并设置过期日期为关联2020年1月的ECA CRV，ECA然后提供给客户端或是Update=7或者新的2020年1月1日的CRV，0000 0001，而客户现在拥有2020年1月1日ECA CRV的版本1。2019年2月22日，客户端收到一个证书，其中RID对应于2020年1月1日ECA CRV, RN为2，检查CRV中的对应位，确定它未被设置，然后继续使用证书。2019年2月22日，RN为2和4的证书提交撤销申请，ECA更新CRV，并向客户端发送0010 1001，或[2,4]。在2020年1月2日，CA ECA和客户端清除了所有的有关2019年2月的库存。

![image-20200909094516737](/images/2020-09-15/image-20200909094516737.png)

### B. Design Analysis

- **对Revocation IDs and Revocation Numbers的评估** RID中的RN时候小的且唯一的，其次有效的包含了过期日期。
- **对Certificate Revocation Vectors的评估** 检查证书撤销情况所需的计算和网络带宽较低，在常数时间便可完成，所需的存储空间较小（100M张证书大概需要12.5MB空间大小）。

而对于证书的更新，Let 's Revoke支持三种用于更新的方法，并且一般选择最小化带宽成本的方法。

![image-20200909105800768](/images/2020-09-15/image-20200909105800768.png)

​	作者还探讨了多个CA设置CRV之间的影响，结果表明，他们之间不会产生影响。

![image-20200909114330609](/images/2020-09-15/image-20200909114330609.png)

### C. Distribution Methods
作者结合之前提到的CDN方法，将CA将CRV信息部署到了CDN上。客户端可以从CDN上下载到最新CRV信息。作者也声明了物联网设备可以基于这种基于拉的方法获得CRV。

## COMPARING REVOCATION STRATEGIES

### A. Efficiency

作者的评估标准主要是设备存储和网络。结果如表二所示。

![image-20200909120346874](/images/2020-09-15/image-20200909120346874.png)

图3总结了所有撤销百分比的存储需求，而图4显示了在日常使用中可能发现的撤销百分比较低时的比较。

![image-20200909125331613](/images/2020-09-15/image-20200909125331613.png)

### B. Timeliness

这里的及时性指的是证书被吊销到客户端被看到的延时，基于拉的策略（例如CRL和OCSP）通常会缓存吊销状态信息，并假定其有效期最多为7天，这段时间可能会被攻击者利用来攻击。而CRLset，OneCRL和CRLite可以将每日更新推送给客户。

### C. Failure Model

这里soft指的是当撤销信息被缓存时，客户端会下载撤销的列表，此时如果网络被劫持，则用户不能够及时下载，将大致网页无法被访问或延迟访问。

### D. Privacy

OCSP不会保留用户隐私。OCSP请求与使用中的证书相对应，为第三方和窃听者提供了用户的浏览习惯。 

### E. Deployability

CRLs，OCSP和CRL Set已经被部署。

### F. Auditability

可审核性用于防止错误的吊销状态。大多数方法都是可审核的，但是，由于CRLSets和OneCRL的性质，无法对吊销策略进行完整性，修改或模棱两可的审计。

## VIABILITY SIMULATIONS

为了进一步验证CRV的性能，作者进行了一些吊销模拟，以表明CRV可以很好地用于当前的日常吊销检查，并可以扩展用于大规模吊销事件和将来更大的证书空间。

​	作者使用Censys.io收集了截至2018年3月21日的全球证书空间的相关吊销数据。作者首先删除了重复的，过期的，私有的，无效的证书来过滤此数据集，保留了8410万个证书。其次，作者将具有CRL节点的证书与仅具有OCSP节点的证书分开。 作者确定了475个没有任何吊销终结点的不可撤销证书，并将它们也从数据集中删除。最后，作者扫描了其余证书以确定它们的吊销状态。

![image-20200909132501842](/images/2020-09-15/image-20200909132501842.png)

​	作者创建了一个模拟器，其参数控制着CA的数量，CA记录的失效日期的数量，每天颁发的证书的数量，每个CA撤销的证书的百分比以及每个CA在给定日期的新吊销的百分比 。

![image-20200909134656101](/images/2020-09-15/image-20200909134656101.png)

## SECURITY ANALYSIS We

​	作者进行了安全性分析。作者假设了一个威胁模型，活跃的网络攻击者可以在其中创建，修改和阻止消息。 攻击者有两个目标：(1)强制客户接受已撤销的证书;(2)强制客户认为有效的证书已被撤销。

- **Accept a Revoked Certificate** 攻击者可以通过阻止客户端更新CRV并得知证书已被吊销，来迫使客户端接受吊销的证书。

- **Revoke a Valid Certificate** 攻击者还可以强迫客户相信有效证书已被撤销。其最终效果是阻止客户端使用与证书关联的服务的拒绝服务攻击。


