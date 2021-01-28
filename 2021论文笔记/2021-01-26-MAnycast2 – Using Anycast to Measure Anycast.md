# MAnycast2 – Using Anycast to Measure Anycast 

> 作者：Raffaele Sommese, Leandro Bertholdo, Gautam Akiwate, Mattijs Jonker, Roland van Rijswijk-Deij, Alberto Dainotti 
> 
>单位：University of Twente, UC San Diego
>
>会议：IMC 2020
>
>论文链接：[MAnycast2 – Using Anycast to Measure Anycast](https://dl.acm.org/doi/10.1145/3419394.3423646)

## Abstract
Anycast寻址是一种将相同的IP地址分配给分布式设备的新型地址分配方式，它被广泛的运用到提升Internet服务弹性和性能上。但是与传统的单播、组播寻址不同，传统的部署模型已经无法从地址本身判断它是否是Anycast。现有的检测Anycast over IPv4前缀的方法存在由路由和动态延迟带来的误差问题，因此本文介绍了一种全新的“MAnycast2”的新技术，可以帮助克服这些问题。这项新技术利用anycast平台的vantage points作为探测节点，它既能消除动态延迟所带来的敏感性影响，又能极大的提高测量效率和延展性，作者利用这种方法在3个小时内完成了对ISI攻击列表中IPv4地址的筛查。
<!-- more -->

## 1 什么是Anycast？

Anycast最开始是被用于BGP（区域网间路由）间的路由寻址的。早期Anycast的被成功的运用在13个根DNS服务器之间，以此提高DNS系统的弹性和抵御DDos攻击的能力。到现在包括公共DNS递归解析器(如CloudFlare、谷歌、Quad9和OpenDNS)、云提供商和内容分发网络CDNs和数百个其他互联网提供商ISP所广泛采用。

## 2 为什么需要测量anycast？
对于IPv6引入了anycast地址这种特殊的方法进行anycast寻址，但是对于IPv4来说却是为多个主机分配了相同的单播地址，这使得路由系统和终端用户无法判断一个地址是否是anycast地址。

## 3 现阶段的方法有什么问题？
现在的检测方法主要针对的是anycast IPv4的前缀，由于路由和动态延迟的不稳定性会对检测的准确性带来影响，同时针对检测方法本身测量负载的效率和可伸缩性也具有挑战。

### 3.1 对照组：iGreedy方法
作者使用iGreedy方法作为实验的对照组方法，这种方法是基于对三角向量的RTT测量来判断一组响应是否来源于同一地理源。但是这种方法会给测试平台（PlanetLab）带来巨大的运行压力，比如测量50K的IP（约占待测列表的1%）就需要花费约2000万个成熟的Atlas credits,要探测完hitlist中全部610万个IP地址，意味着至少需要1.22亿Atlas credits。因此需要一个新的、高效的方法能够定期执行anycast的普查工作。

## 4 M-anycast方法
- 作者的工作是建立在De Vires等人之上，他们是利用部署在CloudFlare上的anycast CDN来进行测量的，使用了现有的技术ping来映射出不同的节点。
- ![](/images/2021-01-26/upload_82a2877fe8feb90d1b2655f9918caeb6.png)
- 方法要点：把vantage points（指的是地理位置上最接近接收方的节点，VPs）作为发送方，把待测试的IP地址设置为接收方，最终的目的是通过测量方法测试出目标IP地址是单播地址（unicast）还是任播地址（anycast）。
- 具体原理：当路由稳定之后，相同地理位置下探测响应的IP地址也是稳定的，由于发送方VPs是anycast，所以理论上任何VP发送到目标IP的探测响应都会被最接近响应目标IP的VP节点所接收，当目标IP是anycast时，探测将会到达最近目标IP的anycast部署节点上，也就是说如果路由是稳定的，对目标IP的所有探测响应都只在一个VP实例上所接收到，如图-1中的蓝色节点，那么目标的很可能就是单播，如果目标IP的探测响应在多个VP实例上所接收到，如图中的红色节点，那么目标很有可能是anycast。

## 5 不同检测方法效果的比较

作者主要比较了M-anycast2和igreed、operator三种

验证方法：

1. 基本验证：对于一些常见的网络基础设施如DNS根服务器和公共DNS解析器进行验证，因为DNS是一定使用了anycast寻址的，由此可以判断三种检测方法是否有出现假阴的情况；
2. AS提供商进行判断：选取了Internet排名前十的anycast提供商，他们的节点应该都被认为是anycast的节点，判断三种方法验证是否准确。

结果：
![](/images/2021-01-26/upload_cd17e35237f83954720a6191accbfbff.png)


## 6 M-anycast2存在的局限性
1. 实验发现了该方法对两个典型的anycast服务存在错误分类，它把C-ROOT和谷歌公共DNS都分类成单播，但显然他们应该是anycast的；
2. 当只有2-3个VPs收到响应时，该方法会提供不明确的结果。

分析：本方法最重要的影响因素是VPs与待测的IP节点之间的连通性，其中关于连通性最小的条件是至少有两个不同的VPs偏好不同的转发路径——然而这个条件在实际的网络环境中并不总是成立的，由此可能导致目标节点如果是采用了anycast寻址，不同的响应流量会被导入到同一个VPs上，从而造成了M-anycast的误判。

## 7 实验结论
该方法具有较低的（0.1%）的假阴性率（anycast地址被错误的归类为单播），而且如果收到的超过4个VPs的响应，则假阳性率较低或为零。但是当只有2-3个VPs收到响应时，该方法会提供不明确的结果，而且对于两个突出的anycast服务（C-ROOT、谷歌公共DNS）采取了错误的划分。