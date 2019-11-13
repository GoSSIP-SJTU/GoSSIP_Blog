---
layout: post
title: "Understanding cross app remote infections on mobile webviews"
date: 2018-12-10 17:43:51 +0800
comments: true
categories: 
---

作者：Tongxin Li, Xinhui Han, Xueqiang Wang, Mingming Zha, Kai Chen, XiaoFeng Wang, Luyi Xing, Nan Zhang, Xiaolong Bai

单位：Peking University, Indiana University Bloomington, Institute of Information Engineering, Chinese Academy of Sciences, Indiana University Bloomington, Tsinghua University

出处：CCS 2017

论文：https://www.informatics.indiana.edu/xw7/papers/p829-li.pdf

<hr/>

## 摘要

跨应用的URL调用在提升用户体验的同时，引入了一定安全隐患，使得远程攻击者可以通过恶意网页窃取应用的控制权，这种威胁被称为XAWI(Cross-App WebView Infection)。它使得一种全新的多应用串通攻击成为可能，这样攻击者可以利用多个被感染的应用逐步提高自己在此移动设备的权限，或进行钓鱼攻击。 

检验发现Google Play最流行的应用中7.4%的都可能遭受XAWI攻击，包括Facebook、Twitter、Amazon。论文指出了跨应用沟通这一提高用户舒适度的需求与对通道进行安全控制之间的矛盾，为了减轻安全威胁提出了一种系统层面的保护方法。

<!--more-->

## 1 XAWI攻击样例

-----------------

### Evil twin from within

利用运行在Chrome中的脚本，可以隐蔽地改变Twitter应用的状态，使用它的WebView模仿自己的登录界面

![avatar](/images/2018-12-10/2.png)

步骤如下：

- Chrome通过scheme调用向Twitter发送一个导航请求。

- Twitter的***UrlInterpreterActicity（public activity）***收到URL后，启动一个新的activity并且将后者的WebView导航到攻击者的网站（使用含有Twitter的package name 的Intent）。

- 打开多个Twitter WebView实例 ，保证第一个位于twitter的任务栈栈顶。
- 向Chrome发送导航请求（googlechrome://)，使Twitter的 WebView被藏到后台。
- 用户之后启动Twitter时将看到钓鱼网页，输入的用户名和密码将被远程攻击者获取。
- 之后被感染的WebView转换到Twitter的主程序，避免用户发觉。

为什么选择Twitter作为攻击对象？

因为Twitter的Webview不含有任何title bar 或者其他UI 部件，较容易创造一个假的登录界面。

### 未授权的应用安装 

借助Amazon Appstore 的WebView的JS 接口提供一个名为 IntentBridge的类， 可以进行app的安装。

困难：

- 它的WebView没有暴露任何UI，让用户导航到non-Amazon domains。

- 管理WebView的活动没有Intent filter，无法使用Intent触发。

- 该应用强制将https://mas-ssr.amazon.com 添加在WebView被要求去访问的URL之前。

![avatar](/images/2018-12-10/5.png)

步骤如下：

- Chrome访问了恶意网站。

- 使Chrome向Amazon Shopping 发送 Intent 将后者的WebView导航到攻击者网站。

  可以做到这一点，是因为Amazon Shopping 使用***Uri.getHost（ ）***来得到URL的域名，而对于形如  https://a:a@test.amazon.com:a@attack.com 的网址，它解析出来域名为test.amazon.com，是符合Amazon规定的，但是这个网址的域名在WebView中被认为是attack.com

- Amazon Shopping 可以利用deep link 来感染Amazon Apptore的WebView。

  - 利用Deep link 指的是：Amazon Shopping 可以将 ***intent:.attacker.com#Intent;package=com.amazon.venezia;component=com.amazon.venezia/com.amazon.venezia.Venezia;end; ***这样的URI转换成为一个package name 为 com.amazon.venezia、activity name 为com.amazon.veneia.Venezia的explicit Intent，如此即便Amazon Appstore 没有Intent filter也可以触发它的活动。

  - 由于Amazon Appstore在添加https://mas-ssr.amazon.com 时末尾不会加/，因此攻击者只需要再构建一个 mas-ssr.amazon.com.attack.com 的子域名作为攻击页面，请求将WebView导航到***.attack.com***即可使之打开攻击页面

- 攻击者可以在后台进行软件安装

## 2 XAMI 介绍

一旦Chrome的WebView中的脚本触发了一个URL，如**fb://webview/?url=[web.page.url]**，Chrome即向Facebook发送一个含有web.page.url的Intent，之后Facebook自动将它的WebView重定向至该web.page.url，加载链接中的内容。这种合作被称为XAWN（Cross-app WebView navigation），利用它可以在不同应用的用户界面实现无缝转换。

而导致XAWI的根本原因在于：XAWN允许由某一应用的WebView加载的Web内容发出导航请求（例如 URL scheme），以启动另一个应用的WebView，使它访问恶意网站。

研究发现，在XAWI攻击中，攻击者可以长久而隐蔽地控制被感染的应用，并继续寻找设备上其他有此漏洞的应用。在它们的基础上，攻击者能够进行复杂的串通攻击，包括一系列从未想到的远程深度钓鱼攻击。这种攻击足够隐蔽，因为并不需要目标机器上装有恶意应用。

## 3 XAMI 基础知识

### 3.1 Activity and Task

Android中，一个WebView依附于一个activity。Activity是一个为用户提供与应用交互界面的组件，一个典型的activity在应用的Manifest文件中使用\<activity\>标签描述。

#### 活动启动模式

一个活动有四种启动方式，不同的启动方式将影响它的WebView实例的运行状态。  

standard模式或singleTOP模式的activity可以被实例化多次；而singleTask模式或singleInstance模式的activity一次只能有一个实例。  

Google官方认为前两种是大多数activity的正常启动方式；而后两种则被称为专业启动方式，不推荐使用。

#### 任务和返回栈

当一个activity在同一设备上调用了其他activity时，之前位于前台的activity将被移至后台并被新activity覆盖，并总保持之前位于前台的activity处于后台首位。Android将这一系列活动链接成为任务（task），并把它们放在任务的返回栈中，方便任务结束或用户按下返回键后回调先前的activity。

### 3.2 Intent 、Intent filter and Deep linking

#### Intent and Intent-filter

为了通过网页内容调用应用的activity，WebView请求托管应用构建一个Android Intent，并通过***StartActivity ( )***发送。

如果Intent含有接受者的package name 和 activity name，操作系统可以直接找到目标组件，此之谓explicit Intent；而implicit Intent不指定确切的应用，由系统与各应用声明的Intent filter中的特性(action，category，data URI）匹配，符合Intent filter则可被此activity执行。当某一Intent匹配多个目标activity时，系统交给用户做出选择。

#### Deep link

为了可以调用其他app中的activity，提出了deep link。

但是deep link是由单个的应用供应商提供的，开发者要使用自己设计的WebViewClient来处理来自WebView的调用，使它满足deep link 协议，而此协议不同应用供应商彼此不同。

它的存在使得WebView接收到某些URI后，可以直接确定package name 和 activity name，到达要触发的activity。

## 4 目标找寻和保护

设计了ViewFinder来自动搜寻易受攻击的应用，与此同时提出一个系统级别的解决方案。

### 4.1 自动XAWI分析

应用易被XAWI攻击的关键———是否有WebView实例被暴露在外，并且可以通过URL远程调用（explicit、implicit、deep links)。

#### 设计

目标：寻找与Intent filter和public activity代码中的URL组件相似的部分URL或字符串，来构造Intent。  

使用基于ADB的fuzzer实现，从与individual activity相关的应用数据中收集线索，将之转换成Intent。监视器监测加载URL后的实例的操作来判断是否成功。

#### URL-guided fuzz

首先查找manifest文件中的public activity（例如"***exported=true***"），通过构造包含合适URI的Intents来检测它们。

> ##### URI field 构造方法
>
> ###### 有Intent filter 的activity
>
> 首先尝试从activity的Intent filter中提取数据片段，特别是那些期望HTTP链接的activity，因为它将在filter中声明域名和路径，可直接被利用。写入恶意的URL后，将之加载到WebView中，检测URL是否被目标activity打开。  
>
> 若activity声明自定义scheme（如：fb://），尝试将目标URL直接加在scheme后作为一种构造，并在清单文件中寻找带有"**?url=**","**?redirection=**","**?uri=**"等的字符串，在它们后面加上目标URL进行构造。
>
> ###### 没有Intent filter的activity
>
> 这些activity需要含有activity name以及正确的data URI的Intent来触发：在应用的代码中寻找那些没有使用HTTP scheme但含有"**?url=**","**?uri=**"等的字符串，使用它们来填充URI域，构造Intent。

fuzz还需要考虑activity在导航WebView时，是直接从URI field读出，还是从extra field读出。由于得到extra field 中的值还需要准确知道它对应的“键值”名，较难实现；且这里只是为了检测应用通过URL被远程调用的可能性，不必关心具体如何实现。因此这里监控系统函数**_Intent.getStringExtra( )_**，一旦它被调用，就将返回值修改为目标URL。

#### 监控

使用开源工具Xposed监视测试用的URL是否被WebView实例加载。

#### 讨论

ViewFinder没有引入false positive。

虽不能保证检验出全部受威胁的应用，但是即使是这种简单的方法也可以很容易地找到大量对远程攻击者有价值的应用。

### 4.2 发现

#### 建立

从Google Play 排名最高的应用中搜集了5000个可以接收来自其他应用的URL方案或Intent的app，涵盖各种分类。  
使用ViewFinder检测，在此基础上有进行人工核查，证实确实没有引入false positive。

#### 概况

5000个应用中，372个（7.4%）被发现无法免疫远程感染，它们的平均下载量为46195505，这将会影响数以万计的用户。且这些应用很多是刚发布的新版本，可见XAWI尚未被应用生产商意识到。

#### 攻击的可能性

- 包含Best Buy、WPS Office和Cymera在内的81.6%的流行应用都可以在后台响应远程命令，使得攻击者可以持续控制应用
- Pinterest、KaKaoTalk和Hola Launcher等都被检测到JS接口、HTML5支持等。它们是RDP的理想材料。  

- 287个应用至少含有一个没有address bar的易受攻击的WebView
- 151个应用的WebView没有title
- 80个应用可以全屏显示网页，它们可以作为实现RDP的帮手（就像Twitter的角色一样）

### 4.3 减轻威胁

由于应用开发者可能暂时无法提供有效的保护，故先提出一种系统级别的解决方案，NaviGuard。

#### NaviGuard

确认并控制那些异常的XAWN，使它们对用户可见。  

借助***StartActivity( )***检测新的activity启动，钩住所有的JS接口以及WebView回调函数，NaviGuard在表格中记录线程编号，以及每个JS接口、回调的WebView编号，以及WebView activity的状态（是否位于栈顶），API调用结束后移除。  

过程如下：

- 一个Intent和它的***StartActivity( )***被发现
- 这个调用是否真的来自一个WebView实例 
  - 这个实例和它对应的activity是否在前台运行
    - 若不是，则终止启动请求
    - 若是，则尝试将这个操作与用户事件相对应 
      - 若失败，则向用户提示此activity  

#### 评价

挑选6款易受攻击的应用（Facebook、Twitter、Baidu等）和44个随机从VeiwFinder报告的应用中选出的app，在NaviGuard的保护下，之前的攻击都将被用户看到，且NaviGuard不会引入明显的时延，开启应用的时间大约只增长0.5%。
