---
layout: post
title: "Geo-locating Drivers: A Study of Sensitive Data Leakage in Ride-Hailing Services"
date: 2019-02-19 13:50:06 +0800
comments: true
categories: 
---

作者：Qingchuan Zhao, Chaoshun Zuo, Giancarlo Pellegrino, Zhiqiang Lin

单位：The Ohio State University, CISPA等

出处：NDSS 2019 

原文链接：https://publications.cispa.saarland/2757/1/ride-hailing_ndss19.pdf

----
#### 文章概述
近几年，Uber等打车软件越来越流行。打车软件的用户包含乘客和司机两个群体，已有的工作大都关注乘客的隐私泄露问题，但对司机的隐私泄露问题却鲜有关注。在这篇文章里，作者选取了包括Uber、Lyft在内的20款流行打车App，重点关注这些App的 __"附近车辆"(nearby cars)__ 功能，以研究司机的隐私泄露问题。研究结果表明，通过对 __附近车辆__ 功能的滥用，攻击者可以获取到司机相关的大量数据，并依此以相对高的正确率确定司机最常去的地址(家)和日常驾驶行为。同时，攻击者也可以从数据中推断出一些打车软件的商业信息，包括车辆总数、车辆使用率等。在此基础上，作者提出了一些防护措施，以保护司机的敏感信息不被泄露。

<!--more-->

#### 相关背景
##### 打车服务(Ride-hailing Service, RHS for short)架构
打车服务的使用方式大致如下：乘客输入目的地后发送订单请求，App搜集设备的位置信息并传输到服务端。随后服务端负责将订单分发到附近的司机处。如果司机接受订单，那么服务器会向双方发送一些附加的信息(如司机预计到达时间等)。通常，打车服务的架构由三个组件构成：
+ 乘客App(Rider App)
+ 后端服务器
+ 司机App(Driver App)

三个组件之间通常以web APIs(基于HTTP/HTTPS协议所定义的接口)方式通信，下图所示为五种常用功能的web APIs示例，包括司机实时位置收集、登录、刷新Token、附近车辆、发送订单请求。除了这五个常用功能，App中可能还集成了其他第三方服务API(如谷歌地图)。
![](/images/2019-02-19/1.png)

##### 研究动机
Pham等人提出了一种攻击，攻击者是一群出租车司机，他们可以通过滥用请求订单的API(Request Ride API)的方式不断发送订单、取消订单获取RHS司机的个人信息。目前RHS提供商已经注意到这种攻击方式并采取了相关措施缓解这一攻击，但作者发现RHS标配的"附近车辆"功能也存在安全隐患，可能会直接、间接地导致司机信息的泄露。另一方面，有报道表明暴力攻击已经成为RHS司机的一个真实威胁，有必要关注司机的隐私信息泄露问题。所以在这篇文章中，作者系统地研究了"附近车辆"功能的安全性，并发现这一功能对司机、平台都会造成严重危害。具体威胁模型为：攻击者既可以是个人，也可以是一群人。此外，攻击者有能力对RHS的App进行逆向工程、创建虚假账号、进行GPS伪装、控制机器连接上互联网。

#### Web API逆向工具
既然要研究"附近车辆"功能的安全性，就需要先找到相关的web APIs。然而作者选取的20款App中只有Lyft对这一功能的API有文档描述，其余App都未公开API。因此，需要设计并实现逆向web API的工具。
##### RHSes 选择
作者选取20个RHSes App的规则：在Google Play Store上搜索ride-hail关键字，并选择前二十款可以安装、运行在测试机上的App。针对这20款App，作者发现其中12款都被混淆处理，同时只有Uber进行了HTTPS证书的检查。

##### Lyft API示例
下图所示为Lyft App的"附近车辆"功能相关的API，包括登录、Token更新、请求附近车辆信息。从这个示例中我们可以明确工具需要解决的三个问题：
+ 精准定位感兴趣的API(请求附近车辆信息的API)
+ 识别API的依赖关系(如下图所示，请求附近车辆信息的API需要Token作为参数)
+ 绕过App的混淆以及证书检查(Uber)
![](/images/2019-02-19/2.png)

##### 工具设计
为了解决上述问题，作者选择使用动态的污点分析的方式来明确App中的信息流(如GPS信息和Token是如何在Web APIs中被定义、使用的)进而定位到相关API。具体分为三步：
+ 将相关Android和系统API的调用存储在日志中，包括名称、参数、返回值，包括HTTP(S)系统库、Socket APIs和打车服务需要的系统API(LocationManager.requestLocationUpdates(), LocationManager.getLastKnownLocation(), System.currentTimeMillis()等)
+ 将每一对HTTP request、response对应起来，并解析URL、request参数、response值，用解析后的结果替换log文件中关于网络收发的API。
+ 对log文件中的API进行解析，首先搜索文件中位置信息相关的系统API的返回值，如果它在某一Web请求的参数中被使用，该Web请求便是待定的Nearby Cars API。更进一步的，如果该请求的response中也存在GPS坐标，那么就将其标定为Nearby Cars API。(如何确定response中包含GPS坐标?)在找到Nearby Cars API后，对其参数向上回溯，最终确定使用"附近车辆"功能所需API的集合。

##### 工具实现
作者基于Xposed这一Hook框架实现了该工具，同时使用了urllib，zlib，json，xml等Python库。除了上述功能，工具还有一个大规模爬取信息的功能，即根据上述Web API逆向的结果，向RHS服务端发送大量"附近车辆"请求并保存回复信息。

#### API安全性分析
利用工具作者已经可以确认20款App的"附近车辆"功能的API，进一步地，作者想要确认在服务端是否存在一些安全机制，防止攻击者滥用该API。为了进行相关实验，作者通过查阅App公开文档以及Web开发最佳实践，确定了如下服务端可能存在的防护措施：
+ 速度限制(Rate Limiting)
    限制某一时间段内访问次数，通常用来抵御Dos攻击。作者将其细化为
    + 每秒的访问次数，该部分信息通过查阅App官网资料获取
    + 同一Session切换IP后是否需要重新认证
+ Session管理
    + 身份认证，即调用Nearby Car API是否需要用户登录
    + Session有效期
+ 防GPS伪造
+ 匿名性
    + 标识符有效期，调用Nearby Car API返回的车辆信息里包含车辆的标识符
    + 个人身份信息，包括姓、名、邮箱、电话等

实验的结果如下图所示。
+ 速度限制
    Uber、Lyft都在文档中说明存在请求速率的限制，但没有明确阈值。Taxify和eCab在实验一段时间后出现速度限制，可能被误以为是Dos攻击。
+ 身份认证
    有14个App需要登录后才能调用Nearby Car API，另外6个不需要
+ Session有效期
    实验中仅有三个App要求更新Session。
+ 防GPS欺骗
    20个App都没有做相关的保护。
+ 标识符有效期
    实验中17个App的车辆标识符从未更改。
+ 个人身份信息
    6个App会泄露附近司机的个人信息

总的来说，这20款App的服务端没有对附近车辆功能相关的API进行专门的保护。
![](/images/2019-02-19/3.png)

#### 实施攻击
##### 数据收集
作者利用伪造的GPS坐标将监视器覆盖整个城市(依照监视器覆盖的面积将城市划分为网格状)，并开始以恒定的速率发送"附近车辆"请求，大多采用四次一秒的速度。针对Lyft，Uber，Heetch，Gett和Flywheel，作者收集了28天，针对Taxify和eCab，作者收集了7天，其余的为15天。获取到数据后，作者对数据进行了一些筛选，只选择全职司机作为研究对象(出现天数超过实验总天数的一半)。

##### 攻击一：追踪司机的日常路线
将监视器的数据汇总后，可以得到司机的日常路线。下图(a)所示为Heetch司机一天的热力图，通过将所有Heetch司机的路线重叠得到，(b)所示为某一Gett司机的路线图。此外，通过对同一司机长期数据的分析，可以推测该司机的工作模式(何时开始工作等)、最常出现的地方等日常行为信息。
![](/images/2019-02-19/4.png)

##### 攻击二：发现司机的雇佣状态和偏好
存在司机同时为多个RHSes工作的现象，通过获取到的数据可以对这一现象进行分析。作者选取了Uber和Lyft司机，尝试寻找匹配的对象。将某一时间不在同一区域的首先排除，并在剩下的数据里选取时间(2秒之内)、空间(60米之内)都匹配的情况。最终，作者在835个Lyft司机和1328个Uber司机中发现401个交集。作者随机选取了100个司机，并将他们的线路显示在地图上连起来，没有发现冲突的情况。

##### 攻击三：商业信息泄露
通过对数据的分析，还可以获取活跃司机数、司机等待时间等不公开的商业信息。作者还对eCab和Taxify的活跃司机数量进行了验证，证明了实验所得数据和其官方的声明是一致的。

#### 防护措施

+ 限制API访问速度，可以缓解攻击
+ 关联性
    攻击者能够对收集到的数据进行分析的前提可以将司机和他们的路线进行关联。虽然14个App都没有直接给司机分配ID这样的标识符，但在response中还是会间接地存在可以等同于司机ID的标识符存在。这也使得攻击者可以将司机和他们的路线进行关联。因此，一个有效的措施便是取消response中的标识符。
+ 人造数据
    作者提出可以使用人造数据以防止商业信息的泄露，但可以导致用户体验不佳。
+ 实现逻辑
    针对6款泄露附近司机个人信息的App，作者提出应当修改App逻辑，保证司机的隐私信息只被需要的人收到。

#### 总结
本文中作者针对20款打车软件的"附近车辆"功能的安全性进行了研究，切入点很小，但从中确实挖掘出了问题，此外内容非常具有逻辑性。但文章中缺乏测试司机标识符有效期的方法以及在司机标识符会发生变化的情况下如何将司机与路径相关联。

