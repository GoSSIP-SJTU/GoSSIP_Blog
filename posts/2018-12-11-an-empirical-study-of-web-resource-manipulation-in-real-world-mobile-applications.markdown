---
layout: post
title: "An Empirical Study of Web Resource Manipulation in Real-world Mobile Applications"
date: 2018-12-11 15:47:17 +0800
comments: true
categories: 
---


出处：USENIX Security 2018  

作者：Xiaohan Zhang, Yuan Zhang, Min Yang, Xiaofeng Wang, Long Lu, Haixin Duan

单位：School of Computer Science, Fudan University, Shanghai Institute of Intelligent Electronics & Systems, Shanghai Institute for Advanced Communication and Data Science, Shanghai Key Laboratory of Data Science, Fudan University, Indiana University Bloomington , Northeastern University , Tsinghua University

原文链接：https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-zhang_0.pdf

----
#### 文章概述
如今，移动App已经成为用户访问Web服务的主要方式。安卓和iOS都提供了丰富的操作Web资源的API，方便开发者在App内部集成Web服务。之前一些学者的工作已经指出了这些API存在的攻击面，也提出了一些防御的机制，但它们都没有给出实际场景下的攻击案例，也没能评估这类攻击所能造成的影响，而这正是这篇文章关注的重点。文章中，作者定义了Web资源的cross-principal manipulation(XPM)问题，实现了XPMChecker这一自动化分析工具，并使用它对Google Play上80694个App进行分析，发现49.2%的Web资源操作都属于XPM，4.8%的App中存在XPM行为，并且超过70%的XPM行为都是针对一些著名的站点。此外，作者还发现有21个App存在明显的恶意行为，如窃取、滥用用户的Cookies、收集用户登录凭证、伪装用户进行登录等。作者也确认了类似的威胁在iOS App中也是存在的，而大多数的Web服务网站尚未意识到这种威胁的存在。

<!--more-->

#### 操作Web资源(Web resource manipulation)
##### 1. WebView
在移动平台，开发者将不同的Web服务集成到同一App中，可以给用户提供更丰富、强大的功能。为了方便集成，Android和iOS都提供App内置Web浏览器的功能来呈现Web内容。这种浏览器在安卓端就是WebView，在iOS端则为UIWebView/WKWebView(下文统称WebView)。在WebView的基础上，移动平台还为开发者提供了丰富的操作Web资源的API来提供更多的功能。比如，安卓和iOS都有evaluateJavascript这一API，允许App在Web页面插入JavaScript代码并返回结果。


##### 2. XPM(Cross-Principal Manipulation)
##### A. 安全隐患
安卓和iOS系统对操作Web资源的API缺乏基于来源的访问控制，也就是说只要是来自App内部的代码都可以通过这些API操作Web资源。举个例子，某一App在WebView中加载了“www.facebook.com”
，那么它就可以使用evaluateJavascript这一API在Facebook界面运行JavaScript代码，获取用户数据。
通过一个实际场景的例子，我们可以了解对Web资源缺乏访问控制所带来的安全问题。如下图所示，有两个App A和B。A是Facebook官方App，B是一个聊天App Chatous。在B中集成了Facebook登录的SDK，用户可以使用Facebook账号登录。在这两个App中有三个Java类(C1、C2、C3)使用了WebView来加载www.facebook.com
页面，同时使用CookieManager.getCookie这一API获取cookies。对于C1、C2，它们读取cookies的行为很正常，但Chatous作为用户、Facebook之外的第三方，也尝试获取用户的cookies，这一点是非常可疑的。通过人工的分析，作者发现Chatous滥用Facebook的cookies收集Facebook用户的信息。
![](/images/2018-12-11/1.png)

##### B. 问题定义
受Web平台同源策略的启发，作者定义了移动平台的XPM问题：定义App所操作的Web资源所属方为WP(Web Principal)，操作Web资源的代码所属方为AP(App Principal)，在某一次App操作Web资源时，如果AP!=WP，则认为是一次XPM，XPM给用户带来安全威胁。作者认为App是不可信任的，其中存在两类攻击者：App本身及其集成的三方库。

##### C. Web资源相关API
基于所操作的Web资源对象的不同，作者总结了安卓和iOS平台常用的四类API，具体如下表所示：
+ 本地存储
+ 网页内容
+ 网址
+ 网络流量
  ![](/images/2018-12-11/2.png)

#### XPMChecker设计与实现
为了实现大规模的App分析，作者实现自动化分析工具XPMChecker，如下图所示，该工具由三个部分组成：
+ 静态分析器
    静态分析器基于Soot和FlowDroid，首先解析APK文件，构建inter-procedure control flow graph(ICFG)，并通过遍历ICFG定位操作Web资源操作点。然后基于观察出的一些模式(下图所示)，提取出所操作的URL、相关代码签名、包名等上下文信息，并存储到数据库中。
    ![](/images/2018-12-11/8.png)
+ 主体标识器
    + Web Principal
        利用URL标识Web Principal，对于短URL和IP地址，动态加载它们或者使用DNS查询。
    + App Principal
        主要是区别代码中主体App代码和三方库代码。使用静态分析器获取的代码签名，并基于三方库代码会出现在不同App中这一发现，对不同的Web资源操作点进行WP和AP的标识。
+ XPM分类器
    确定Web Principal与App Principal是否相同，但存在一些情况，如Web Principal为facebook，而App Princiapl被简写为fb，这些情况下会发生误判。作者将该问题转化为：利用搜索引擎搜索Web Principal和App Principal，利用bag-of-words模型对返回内容进行处理，并通过返回内容的相似性(仅关注每一个单词出现的频率)来确定二者的相似性，并设置相似性的阈值(阈值的设置需要进行进一步的实验)。

![](/images/2018-12-11/3.png)

#### 实验评估
作者从Google Play中下载了84712个App，使用XPMChecker进行分析。运行环境为安装CentOS 7.4 64位操作系统的64核、188GB内存的工作站。实验开启九个进程并行分析，并为每个App设置20分钟的时间限制。所有App的分析共花费233个小时，平均每一App花费10秒钟。XPMChecker的静态分析模块成功分析了80694(95.3%)个App，其余App或者运行超时，或者是Soot和FlowDroid对其分析失败。
![](/images/2018-12-11/5.png)
+ 针对静态分析器，作者人工分析了50个App，发现其中存在36个Web资源操作点，而静态分析器只成功标记了其中的33(91.7%)个，因为复杂的字符串编码、多层的函数调用等原因，静态分析器在分析其余三个Web资源操作点时都没能成功提取URL。
+ 针对主体标识器和XPM分类器，作者选取了1200个Web资源操作点，对于其中的1000个，尝试不同的阈值以取得误报率和漏报率之间的平衡。确定了阈值后，使用剩下的200个进行测试，结果如下图所示：
    ![](/images/2018-12-11/4.png)

#### XPM分类
为了更深入地了解这些XPM行为，作者又人工分析了存在XPM行为的88个库、100个App。其中88个库涵盖了63.6%的XPM行为。
![](/images/2018-12-11/6.png)
通过分析，作者发现：
+ 大多数的XPM行为对于提高移动端用户的易用性都是有必要的
+ 一些XPM行为不安全地实现了OAuth隐式授权(implicit grant flow)
+ 存在一些恶意的XPM行为，作者也选取了200个App进行人工分析，发现其中21个存在恶意的XPM行为，同时安卓和iOS平台都存在这种情况。
    + 在OAuth中模仿可依赖方
    + 窃取用户凭证
    + 窃取、滥用Cookies
+ 恶意的XPM行为影响大量用户，21个App的安装量最高有131220000，最低有29885000。

#### 其他发现
+ 大多数Web服务提供商还没有意识到Web资源操作存在的风险，也不能有效地防止用户访问WebView中的敏感页面。
  ![](/images/2018-12-11/7.png)
+ 完全隔离WebView不适用于大多数的App
+ 操作Web资源的API必须要有细粒度的访问控制

#### 总结
文章重点对安卓平台的XPM行为进行了研究，在自动化分析的基础上也加入了很多人工分析，对XPM行为的安全性研究非常深入。不过可能受限于篇幅，作者在文中经常提及iOS端也有类似的XPM行为，却一直没有给出相关的案例进行进一步的说明与解释。
