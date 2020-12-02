---
layout: post
title: "On the Relation between Outdated Docker Containers, Severity Vulnerabilities, and Bugs"
date: 2019-04-25 16:39:10 +0800
comments: true
categories: 
---

作者: Ahmed Zerouali, Tom Mens, Gregorio Robles, Jesus M. Gonzalez-Barahona

单位: UMONS, Belgium and URJC, Spain

出处: SANER'19

原文: https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=8668013

## INTRODUCTION

本文基于从Docker Hub中提取Docker镜像，识别安装在其中的软件包，并通过分析这些软件包的technical lag来计算镜像的technical lag。 根据版本更新，漏洞和bug的数量来衡量单个软件包的technical lag。 初始样本由Docker Hub中基于Debian的所有官方镜像和社区镜像组成。 因此只需要为Debian软件包计算technical lag。

<!--more-->
作者想要解决的问题：
1. How often are Docker images updated?
2. How outdated are container packages?
3. How vulnerable are container packages?
4. To which extent do containers suffer from bugs in packages? 
5. How long do bugs and security vulnerabilities remain unfixed?

## METHOD AND DATA EXTRACTION


整个流程如下图所示，
1. 从Docker Hub收集基于Debian的基础镜像
2. 识别由基础镜像派生的其他镜像
3. 对镜像中的packages和历史的packages作比较
4. 基于历史的数据库，识别bug和vulnerability

![](/images/2019-04-25/media/15554819593116/15560808769397.jpg)

### Base Images for Debian
作者选择了Debian作为分析的基础，在2018年10月1号，Debian仓库有125M的pull。

Debian同时有多个发行版本，包括Testing, Stable 和 Oldstable。当前的版本如下。

![](/images/2019-04-25/media/15554819593116/15560813888427.jpg)

### Identifying Analyzed Images

举个例子，debian:stretch的Dockerfile

![](/images/2019-04-25/media/15554819593116/15560820094208.jpg)

当创建完后，将生成一层layer

![](/images/2019-04-25/media/15554819593116/15560820909317.jpg)

当其他镜像使用debian:stretch时，就可以使用上述的镜像

![](/images/2019-04-25/media/15554819593116/15560821584468.jpg)

由此生成debian:stretch-backports镜像，并包括debian:stretch

![](/images/2019-04-25/media/15554819593116/15560821951667.jpg)

因此，如果检查Debian的派生镜像，只要检查他们是否包含基础镜像的layer即可
最终从官方的14653的镜像中发现2453个唯一的基于Debian的镜像。从社区的30000个中发现4927个唯一的基于Debian的镜像。一共是7380个。

![](/images/2019-04-25/media/15554819593116/15560819869693.jpg)

### Identifying Installed Packages
首先从Oldstable (Jessie), Stable (Stretch) and Testing (Buster)收集所有的packages，然后在镜像中的使用dpkg -l查看所有安装的package。通过匹配发现99%的package都在收集的package中，其中1,379,163 个属于official images, 561,982 个属于community image。每个official和community安装的package的中位数是190和261。

### Vulnerability Reports
为了识别安全问题，使用了Debian Security Bug Tracker。截止2018年3月18号。对于每个package，包括受影响的版本号，危害程度，影响的发行版本，修复的版本等等。

### Bug Reports
对于错误报告，使用Ultimate Debian Database。 UDD是一个不断更新的系统，它收集了各种Debian数据：错误，包，上传历史，维护者等。

## EMPIRICAL ANALYSIS RESULTS
1. How often are Docker images updated?

如图显示了上次更新所考虑的Docker镜像的年份。 对于使用Oldstable版本Jessie（Debian 8），2018年更新的官方镜像的数量少于2017年更新的镜像，而社区镜像则相反。 另一个重要观察结果是，48％的社区镜像和66％的官方镜像都是2018年之前更新的。

![](/images/2019-04-25/media/15554819593116/15560848545307.jpg)

结论：
*   超过一半的Docker镜像近四个月未更新。

2. How outdated are container packages?

下图显示了基于Debian版本的官方和社区Docker容器中最新和过时软件包的比例。 可以观察到，无论Debian版本如何，大多数软件包都是最新的。 每个容器的最新包的中位数比例是所有已安装包的82％。 我们还注意到社区容器内的包（中位数为85％）比官方容器内的包（中位数为78％）略微更新。

![](/images/2019-04-25/media/15554819593116/15560854993935.jpg)

下图显示了Docker容器中过时软件包版本的滞后情况。使用Stretch的容器明显是高度偏斜的。

![](/images/2019-04-25/media/15554819593116/15560854911912.jpg)

下表显示Jessie和Stretch容器的中位数版本滞后为1，而Buster则为2个版本。 这个小差异与Debian发布的状态有关。 由于Buster现在处于测试阶段，很难跟上其更新过程。 但是，我们可以得出结论，一般来说，Docker容器中的软件包要么是最新的，要么落后于中间版本的1到2个版本。

![](/images/2019-04-25/media/15554819593116/15560854752465.jpg)

80％的已使用Stretch软件包版本和90％的Jessie软件包版本是在2017年6月18日之前创建的，这是Stable版本的发布日期。 63％的Jessie软件包版本是在Jessie发布之前创建的（即2015-04-25）。 

结论：
*   容器中1/5的安装包是过时的。
*   用户倾向于使用最近更新的官网镜像。
*   过时安装的软件包落后一到两个版本。

3. How vulnerable are container packages?

12.2%（488/3975）的package存在安全问题。
下图展示了不同危害的安全问题的程度分布，包括not assigned, unimportant, low, medium 和high。以及当前的状态，包括open, resolved 和 undetermined

49.9%的漏洞已经修复，48.6%的仍存在，1.6%还未确认。
漏洞主要由medium (37.2%), unimportant (20.2%) 和 high (18.3%)组成

![](/images/2019-04-25/media/15554819593116/15560880916461.jpg)

下表定量展示了容器中存在的漏洞数量。

![](/images/2019-04-25/media/15554819593116/15560881011206.jpg)

过时的容器包与容器的脆弱性之间的关系，比较了过时的包的数量和每个容器的漏洞数量。 直观地观察两个指标之间的关系，特别是对于Jessie容器：当过时的包的数量增加时，漏洞的数量也会增加。

![](/images/2019-04-25/media/15554819593116/15560881077520.jpg)

下表显示了漏洞数量（#vulns）前5的易受攻击的package，已安装软件包的数量（即#pkgs）和时间。 

![](/images/2019-04-25/media/15554819593116/15560881760017.jpg)

下表显示了前5个最易受攻击的源程序包，其中包含漏洞数量和使用它们的容器的比例。 


![](/images/2019-04-25/media/15554819593116/15560881903119.jpg)

结论：
* 近一半的漏洞没有被修复。 
* 所有的容器都存在高危漏洞。
* 漏洞的数量取决于所使用的Debian版本。
* 最易受攻击的容器超过2年没有更新。
* 容器中过时的软件包数量与已修复的漏洞数量密切相关。

4. To which extent do containers suffer from bugs in packages? 

问题涉及Docker容器包中存在与安全无关的错误，以及错误和过时包之间的关系。 考虑到社区和官方图像的所有包，我们发现所有独特安装源包中的50.1％（1,994个中有1,994个）存在错误。 我们还发现所有容器都有“buggy”包。

下图显示了按状态（pending, forwarded, fixed)）和严重性（wishlist, minor, normal, important, high）分组的bug分布。 
65.5% (12,863) 的 bugs 还处于pending, 只有 27.2% (30,922) 的bug被修复了。2.9% 的bugs 是高危的, 27.7% 是重要的, 50.2% 是普通的 normal 。

![](/images/2019-04-25/media/15554819593116/15560935796940.jpg)

下表定量展示了容器中存在的bug数量。

![](/images/2019-04-25/media/15554819593116/15560938875647.jpg)

与我们在漏洞中观察到的相反，容器中的bug数量和过时的package的数量之间存在微弱的相关性。

![](/images/2019-04-25/media/15554819593116/15560938970428.jpg)


结论：
* 所有容器都存在有bug的packages。 
* 安装包中有65％的bug没有修复。 
* Bug的数量与使用的Debian版本有关。
* 容器中的bug数量和过时的package的数量之间存在微弱的相关性。

5. How long do bugs and security vulnerabilities remain unfixed?

由于近一半的漏洞仍处于开放状态，并且所有漏洞中有65％仍处于未决状态，因此这个问题将调查漏洞修复所需的时间。 

![](/images/2019-04-25/media/15554819593116/15560865648379.jpg)

结论：
普通的错误和小错误需要在中位数非常长的时间内修复（53.8和33.5个月）。 高严重性错误修复的速度比其他类型的错误快十倍。

![](/images/2019-04-25/media/15554819593116/15560865539626.jpg)

结论：
高危和中等危害的漏洞比低严重性漏洞修复的更快。 