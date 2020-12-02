---
layout: post
title: "Melting Pot of Origins: Compromising theIntermediary Web Services that Rehost Websites"
date: 2020-05-15 15:20:01 +0800
comments: true
categories: 
---

> 会议：NDSS 2020
>
> 原文：[link](https://www.ndss-symposium.org/ndss-paper/melting-pot-of-origins-compromising-the-intermediary-web-services-that-rehost-websites/)
> 
> 作者：
> 1. Takuya Watanabe NTT Secure Platform Laboratories, Tokyo, Japan
> 2. Eitaro Shioji NTT Secure Platform Laboratories, Tokyo, Japan
> 3. [Mitsuaki Akiyama](https://www.ntt.co.jp/RD/e/organization/researcher/researcher_akiyama.html) NTT Secure Platform Laboratories, Tokyo, Japan
> 4. Tatsuya Mori NTT Secure Platform Laboratories, Tokyo, Japan / NICT, Tokyo, Japan/ RIKEN AIP, Tokyo, Japan
>
> **lab：**[NTT](https://www.ntt.co.jp/sc/index_e.html) 

## Abstract

在本篇论文中，作者研究了wb托管服务——Web代理，Web翻译和Web档案。web托管服务使用一个域名来托管不同域名的网站，由此可能会违反不同域名的同源安全策略。作者首次对此类漏洞进行了大规模研究。作者可以利用此漏洞执行5种不同的攻击：持久中间人攻击；滥用访问各种资源的特权；窃取凭据；窃取浏览器历史记录；会话劫持/注入。作者对21种流行的Web托管服务（每天访问量超过2亿）进行了广泛的分析，发现这些攻击是可行的。 针对此观察，作者针对每种类型的攻击提供了有效的对策。

<!-- more -->

## Introduction

作者本篇论文研究的主要对象是三种web托管服务：*web proxy, web translator, web archive*（Web存档使用户能够访问以前发布的Web内容的某个版本，该版本由于过期、维护或阻塞等各种原因目前不可用。）

web托管服务增强了网络的可访问性。但是由于用户可能会在web托管服务上输入隐私信息，因此攻击者可能会利用web托管服务来获取这些信息。本篇文章是第一次针对具有托管属性的网站进行大规模的研究，并发现常见的安全问题。

攻击的核心想法是，一个有web托管服务提供的域名可以被用来访问多种被托管的网站。这种“melting pot of origins”情况允许攻击者绕开同源安全策略（SOP）的过滤。通过使用这种攻击，作者发现可以展开多种其他的攻击。

作者进一步检查了21种流行的web托管服务并检查是否存在这种漏洞。并发现了有18个web托管服务至少存在一种攻击。

## Background
### A. Web Rehosting in the Wild
图1展示了对web托管服务的流程。
![](/images/2020-05-15/2.PNG)
表一总结了本次工作中研究的21个Web托管服务。（作者在供应商的要求下匿名了两个应用）
![3](/images/2020-05-15/3.PNG)

### B. Advanced Web Features
**Service Worker** Service Worker的一个显着特征是它可以代理Web客户端和服务器之间的所有请求和响应，并修改网页内容。

需要对Service Worker实施强安全限制以防止它强大的功能被利用：
 - 在安全的环境中被操作，如HTTPS或者本地
 - 遵守SOP，确保符合SOP的Service Worker与源和URL路径相关联，并且仅在其路径包含Service Worker脚本的URL上运行。因此，如果一个web服务器环境想通用托管服务一样，通过操作域的子域名或URL路径的不同来分隔操作区域，便不能在同一个Web服务器下运行在不同子域名或URL路径的网站注册一个service worker。
 - 浏览器要求Service Worker脚本的MIME类型仅为JS，text/javascript, application/javascript,  和application/x-javascript; 否则不可使用。

**Application Cache** HTML5标准提供了一套应用程序缓存机制（AppCache ），这允许web应用程序离线运行。有三大优势：Offline browsing 离线浏览，Speed 提升页面加载速度，Reduced workload降低服务器压力。为实现离线浏览，脱机浏览AppCache提供了缓存的替代资源，而不是由于网络或服务器错误而显示的后备页面。很多最新的浏览器支持这项功能，对其的约束与Service worker相似，唯一的不同就是AppCache在同源的页面上工作时，不依赖于其路径；

**Browser Permissions** web浏览器支持访问各种各样的资源，如地理定位，相机，麦克风，和通知，访问这些资源需要用户的许可。在设置此功能时，有一个假设——不同的域名运行着不同的访问资源策略。然而，通过Web托管服务授予的访问权限违反了此假设。

**Browser-based Password Managers** 主流Web浏览器（例如Chrome，Firefox，Opera，IE和Safari）都带有内置的密码管理器。 通常，他们的方式工作如下： 首先，用户访问网站并输入密码以对网站上运行的服务进行身份验证。 然后，浏览器将询问用户是否保存密码。 如果用户允许保存密码，浏览器会将其存储在用户的设备或在云上运行的关联数据库中。 当您以后重新访问该网站时，浏览器会自动填充所存储的密码。由于密码管理器是通过检查域名来识别用户所用密码的，对于web托管服务来说，由于它使用的是单一的域名托管多个网站，这可能会违反密码管理器所做的基本假设，即不同的网站应具有不同的域名。

### Pitfalls of Cookies
在这里，作者列举了攻击所需Cookies的缺陷。
 - **Access from JS** 如果设置了HttpOnly标志，且HttpOnly成功的话，那么JS脚本就不可以拿到Cookie，但是大部分网站这个标志被设置为false。这意味着在受害者访问恶意网站时，其Cookies set就可以被恶意网站的JS脚本得到。同时，由JavaScript编写的cookie可以被另一个JavaScript访问。
 - **Session  Cookie** 如果没有设置过期的日期，那么cookie会变成会话cookie， session cookie会在浏览器关闭后被删除。但是实际上，在浏览器中配置“Continue where you  left  off”设置将使会话保持活动状态。 因此，部署会话cookie并不是针对利用cookie的攻击的对策。
 - **Cookie Bomb** Web服务器拒绝那些请求头太大的请求。如果对于一个网站设置了太多太大的cookies，服务器通常会返回错误信息。
## Threat Model & Attacks
### Threat Model based on Origin Unification
如图所示是托管服务中对不同域名设置了网页。托管服务会将原始的资源转换为具有相同源的页面。当SOP有效时，evil.example是无法获取到a.example和b.example的资源的，但是**托管服务会将页面映射到相同源的**， 此时SOP将不再有用。

攻击过程：
1. 攻击者构造URL为https://rehosted.example/rehost?url=https://evil.example的包含恶意脚本的网页，并使其运行。
2. 攻击者通过传统的Web攻击场景（下载、网络钓鱼和跨站点请求伪造，垃圾邮件，恶意媒体链接的帖子）诱导受害者访问恶意页面。这里需要注意，某些网络重新托管服务会通过验证referrers或HTTP会话来禁止进行热链接（即直接链接到重新托管的页面），这增加了攻击的攻击的难度。
3. 触发攻击

为了使得攻击更加有效，攻击者可以使用一个“登陆页面”，这个页面嵌入的多种iframe标签指向由各种web托管服务代理的恶意页面。通过单独访问这一个页面，攻击者便可实现利用多个托管服务实现攻击的场景。而这个“登陆页面“是不需要被托管的。对于模拟域名有几种选择：通过缩短的URL进行包装或重定向服务，并通过XSS和网站伪造将其注入合法站点。

![](/images/2020-05-15/4.PNG)

### Attacks against Web Rehosting
基于以上攻击模型，作者将典型的Web攻击和Web托管服务结合起来。如下图所示，作者将其分为两种类型：
 - 利用资源来了解哪些网络托管用户之前（即在访问之前）已经读/写过，
 - 从此时开始（即访问后）寄生资源以监视和篡改受害者的浏览器。
![](/images/2020-05-15/5.PNG)
#### Persistent MITM

这是一种新型MITM攻击，在受到off-path攻击者的攻击后，利用Service Worker或AppCache可以持续起作用。 

首先是Service Worker。
作者发现，一个攻击者可以在**web托管服务提供的源上注册一个恶意的service worker**。

在Listing 1中，为一个普通的HTML注册了一个service worker。在这种情况下，根路径下的sw.js被注册为在web托管源路径下的sw.js。攻击者可以在sw.js中实现攻击，该功能可以读取和重写Web请求和响应。 
![](/images/2020-05-15/6.PNG)
Listing2展示了攻击者注册了一个恶意的service worker。
![](/images/2020-05-15/7.PNG)
当受害者访问被托管的恶意页面时，MITM攻击成功。如下图所示。SW可以执行各种攻击方案，例如修改新文章的细微差别，替换电影，注入广告，显示网络钓鱼页面以及用恶意软件替换下载的文件。
![](/images/2020-05-15/8.PNG)
和传统的MITM相比，新MITM攻击能力更大。因为，无需直接拦截网络路径上的通信，service worker注册成功后便会进行永久攻击，并且即使启用HTTPS，页面也会受到影响。 **不幸的是，现代浏览器并没有一个容易被用户理解的方法来检查service worker。**为了检查是否存在恶意的service worker被注册，用户需要打开开发者工具，小心地检查Service Worker的设置。

对于AppCache。

攻击者无需Service worker便可进行持久性MITM攻击。与上面的攻击类似：
 - 敌手首先托管一个恶意的清单文件，然后重新托管一个HTML文件，这个HTML文件包含有托管清单文件的URL。
 - 之后，作者写入fallback规则在清单文件中，需要列出两个URL：第一个是要重写的页面（可用通配符），第二个是备用页面。
    ![](/images/2020-05-15/9.PNG)

 - 注意，这两个源是相同的。虽然，规则显示，只有当返回错误状态码时，AppCache才会启用重写机制。但是可以利用之前浏览器发送给服务器的消息头中包含很大的cookies时，会返回错误码，来触发攻击。在上面的示例中，攻击者甚至通过AppCache篡改了受害者通过Web托管服务访问的所有页面。

下表是两个方法的区别。
![](/images/2020-05-15/10.PNG)

#### Privilege  Abuse
在存在Web托管的情况下，对于硬件的访问权限，如相机或者GPS可能会被到其他的托管的网页使用。图四是用户访问在Wayback Machine上托管的合法网站时位置许可请求的示例。 一旦用户单击“允许”，攻击者稍后便可以使用Web托管服务的恶意页面来转移权限。
此外，攻击者可以通过组合恶意的Serivice Worker和通知权限来分发Web Push通知。在这种情况下，当浏览器程序在运行时，浏览器就会一直获得来自攻击者的Web Push通知。这些通知可以包括钓鱼、恶意图片和恶意网站URL链接的信息。
注意，这种攻击不会影响某些Web托管服务，即在iframe中加载托管页面，并且其域与顶级浏览上下文的域不同。 在这种情况下，iframe的沙盒机制会自动拒绝任何权限请求，而无需与用户进行互动。因此，在这种情况下，攻击失败。

#### Credential Theft
保存在浏览器中的密码的信任凭据与网页的源相关。当浏览器访问某个页面时，浏览器密码管理器的自动填充功能会将与页面源相对应的凭据自动输入到登录表单中。 当受害者访问的是恶意的web托管服务页面时，页面的恶意JS代码就会盗取自动填入登陆框的信息。
注意，浏览器是否填写登录表单，除了与页面源有关，而且与form的设计有关，因此攻击者需要精心设计一个与真正页面相似的表单。

#### History Theft
使用cookie和localStorage的JavaScript代码在现代网站中很常见。 数据分别存储在每个源中，因此，web托管的页面会在cookie和localStorage中共享数据。 攻击者可能使用这些数据来对访问的网站进行指纹识别； 然后，攻击者就会窃取受害用户的浏览历史记录。

数据通常由键值对组成。 Cookie或locakeylStorage中使用的key字符在每个网站中都是静态定义的，并且value字符通常是动态分配的。 因此，攻击者使用进行网站指纹识别。作者凭经验发现某个页面中的localStorage的值以JSON格式存储。如果JSON数据具有关联数组结构，则攻击者可以递归解析它并提取其他密钥以进行指纹识别。

作者发现cookie的失效时间通过是在网站访问时间上加上一定的时间（例如一个月）来生成。 对于使用此类cookie的页面，攻击者还可能从cookie的过期时间推断出目标用户的访问时间。 

#### Session  Hijacking  and  Injection
web代理依赖于HTTP头中的cookies来保证浏览器和原页面间的HTTP会话。Web代理会将新的cookie放在HTTP头中，当用户使用Web代理登录到一个固定的服务中时，浏览器会存储Web代理提供新的cookie。在这种情况下，攻击者可以使用JS窃取此cookie，然后劫持来自代理恶意网页的原页面的HTTP会话。注意到，虽然HttpOnly可以阻止这种情况的发生，但是大多数代理服务并没有使用HttpOnly。

一个攻击者也可以在受害者浏览器中注入一个会话，并且强制受害者登录一个账户。

会话劫持仅适用于通过web托管服务主动登录到Web服务的用户，而会话注入适用于未按自行意愿登录服务的用户。 作者注意到受害者可能怀疑会话注入攻击，因为他们会使用登录到攻击者准备的陌生帐户来浏览服务。

#### Rehosting Rules
由于上述五种攻击依赖于Web托管的规则。作者介绍了常用的web托管规则，以及攻击者如何滥用这些规则来操纵浏览器资源。

1. **URL Rewriting** 最基本的规则就是URL重写规则。Web托管服务主要为托管的页面提供两种类型的URL命名约定：URL查询惯例（使用Web代理和Web转换器）https://rehosted.example/rehost?url=evil.example；UNIX路径之类的约定。https://rehosted.example/evil.example/。对于使用前者的URL重写，service worker可以作用于所有托管页面，攻击成功。另一种则不行。对于URL查询规则，注册的service worker会影响所有重新托管的页面。 另一方面，对于类UNIX路径的规则，除了evil.example页面，注册的service worker无法影响其他页面。

2. **Rehostable File Type** web代理和Web档案服务可以用原始MIME类型托管所有的内容。在这种情况下，攻击者可以放置一个恶意的service worker或AppCache 清单文件在源托管文件中；对于翻译服务，当JS脚本被放置在scr属性和script标签中，那么自动会被托管，以防止内容混乱带来的安全问题。在这种情况下，在src属性中的URL被转换为重定位JS的URL，因此一个攻击者可以使用一个恶意serviceworker脚本放在源托管页面。

3. **Handling  Browser  Resources** 虽然被托管页面的源变为由web托管服务所提供，但是JS代码不会被重写。因此，保存在良性页面JS代码中的资源和许可都可以被恶意页面访问到。
	
	HTTP头中的cookies依赖于web托管属性。Wayback Machine通过向标头名称添加特定前缀（例如x-archive-orig-set-cookie）来禁用cookie存储。其他的web档案托管和翻译服务仅仅丢弃Set-Cookie的头部。Web代理隐式或显式中继HTTP头中的cookie，以便在浏览器和托管页面之间重建HTTP会话。如下图所示。
	![](/images/2020-05-15/11.PNG)
	这两种cookie方法都透明地维护web浏览器和web服务器之间的HTTP会话。作者的攻击可以直接劫持会话或为显式中继cookie的web代理服务注入会话id。对于隐式中继cookie的Web代理服务，尽管原始会话ID被隐藏了，作者的攻击仍然可以劫持会话或注入生成的会话ID。

## FEASIBILITY ANALYSIS
### Vulnerable Rehosting Services in the Wild
作者对于21个web托管服务进行了研究。其中18个服务包含了上述类型攻击产生的漏洞。
3个web托管服务不允许热链接，限制了攻击的可行性。

有三个类型的web托管服务使用了HTTP协议，这种将会受到典型的MITM攻击；有13个服务存在持久性MITM攻击漏洞，12个是由Service worker攻击产生的，这也是作者认为的最强的攻击手段。

作者还发现了一个有趣的现象，谷歌翻译有一个特点，就是会翻译用户上传的文档。如果这个页面包含漏洞，并且存在长久的service worker，那么攻击者可能会盗取用户文档中的信息。

作者还发现一部分网页，没有攻击者可利用service worker进行攻击的漏洞，或者没有使用URL查询惯例重写机制。此时攻击者会转为之前提到的AppCache攻击，并且攻击成功。

作者注意到Google，Yandex，Bing，Baidu和PROMT提供的翻译器将托管的内容放置在iframe中，该内容受沙箱属性保护。 这些服务可以抵御特权滥用攻击，但其余服务容易受到攻击。 作者发现，在所有被调查的Web代理服务中都存在credential  theft  attack的漏洞。 此外，对于所有已启用JavaScript的调查服务，盗窃浏览历史记录是可行的。

对于所有的web代理服务，劫持session和注入攻击者会话是有效的。

对于Weblio和Wayback Machine，一个用于不可以登录一个web托管服务的网页，因为这些服务不算是web代理，用户仍可以登录他自己的服务。这些服务为登录用户提供额外的功：例如，Weblio提供了一本词汇书或一个查看考试结果的控制台，而Wayback Machine允许用户查看上传或收藏的网页列表。作者发现，可以使用类似于Session  Hijacking  and  Injection中描述的过程劫持服务本身的登录会话。

### Evaluation of Fingerprinting
在本节，作者评估了browsing history theft使用的fingerprinting技术的影响。作者评估了三个方面：网站的可识别性；指纹的生命周期；指纹是否泄露了用户的访问时间。
1. **检测指纹可用性的网站** 作者检查了cookie中，本地存储中以及JSON dictionary中是否包含指纹泄露的情况，网站作者选择了Alex 排名Top-10K的网站。作者使用代理站点进行了实验。作者发现，通过作者提出的指纹可唯一识别的网站比例为39.1％（2,541）。作者列出了可指纹网站的前10个类别。该观察结果表明，攻击者可以通过 history theft attack来估计受害者的概况。此外，作者还发现了可识别指纹的网站，包括色情、约会和盗版等敏感网站。
![](/images/2020-05-15/11.PNG)

2. **指纹的生命周期** 为了研究时间对指纹唯一性的影响，作者进行了一项模拟用户访问网站后删除过期Cookie的实验。 作者假设用户访问每个网站一次。
图6显示了网站指纹可用性的变化。 显示了两种情况：所有会话cookie都处于活动状态以及它们何时过期。 当经过时间为零时，可用指纹的百分比为100％（对于活动会话cookie）和96.4％（对于过期的会话cookie）。 在一天的用户访问后，该百分比分别下降到69.4％和64.2％。 不需要长时间存储的Cookie的有效期通常少于24小时。 第二天后下降趋势变得温和，因为具有长寿命的cookie和持久的localStorage密钥有助于指纹的唯一性。
由于一些网站以月为单位设置到期日期，例如1个月、2个月或3个月，我们可以看到在第30天、第60天和第90天出现小峰值。
![](/images/2020-05-15/13.PNG)
3. **Fingerprints 泄露用户访问时间** 由于cookie的过期时间等于访问时间加上一段时间，而访问时间就可以等于cookie过期时间减去增量。这可使得攻击者能够得到用户的访问时间。 结果，作者发现73.6％的指纹泄漏了访问时间。

### **每个浏览器的资源访问行为** 
作者在这里讨论了不同的浏览器的资源访问行为和他们对攻击成功和失败的影响。下表是作者研究的浏览器。
![1](/images/2020-05-15/1.PNG)

1. 对于询问。如果浏览器提示ask once，将会有很大的滥用特权的风险，因为一旦资源访问被允许，它将不会再询问。对于标签为selective，当使用者选择yes时，攻击者才可以开始实施攻击。
2. 对于密钥自动填充。标记为“自动填充”的浏览器会在页面加载时自动使用保存的密码填充密码字段，从而使凭证盗窃变得可行。 而标记为“手动”的浏览器将焦点集中在登录表单元素上，但除非用户明确指示，否则不会填充，在这种情况下攻击不会发生。
3. 关于会话cookie，标记为“keep by  default”的浏览器在退出浏览器时不会删除会话cookie；标记为“keep by config”的浏览器在通过更改配置以保存退出选项，这种情况下不会删除会话cookie。这两种情况下，攻击者将会有一个很长的机会盗取session cookies，由此受害者可能会有很高的收到session劫持攻击和history  theft攻击的风险。

## DISCUSSION
1. Coverage of Our Experiments 作者在这里分析了它检查的web托管服务的全面性。
2. Human factors 在本文中，作者提出了一个通常存在于这些服务中的严重威胁，并证明了许多服务实际上是脆弱的。另一方面，作者并没有从用户实际使用这些服务的角度来考察。但是作者给出了充分的实例，说明他们的假设是成立的。
3. Ethical considerations
## DEFENSES
1. **All Atacks** 由于漏洞的根本原因是，最初设计为放置在不同来源中的Web资源被混合到同一来源中，因此，解决此攻击的直接方法是，不仅使用单独的域名来分隔托管的网站，而且还为每个托管的网站生成一个不同的子域。该解决方案的主要缺点是，对于已经生成的URL（可能会从其他位置引用）无效。该解决方案将特别影响Web存档服务，因为许多存档的网站都是从大量外部网站链接而来的，使得替换此类链接不切实际。将对旧URL的访问重定向到新URL将解决此问题。对于已经托管的网站，可以生成一个第三方无法访问的临时URL。这种情况下，就无法将页面进行共享。
	将浏览器转为私有模式是一种有效的对策，可以在用户端采取这种措施，以完全阻止或减轻作者文中提到的某些攻击。例如，在Edge和Firefox的私有模式下Service worker和AppCache被禁用。它们在基于Chromium的浏览器的专用模式下启用，但在关闭浏览器窗口时被删除。在所有浏览器中，密码管理器均已禁用，并且在关闭浏览器时会删除cookie和localStorage。
	
2. **Persistent   MITM** 需要限制service worker和AppCache行为。Web托管服务器可以通过拒绝浏览器标头中Service-Worker: script具有此头的请求，来阻止恶意Service worker的注册。因为现代的托管服务都没有利用Service Worker工作，因此这种方法不会付出代价。AppCache，攻击利用了HTML标签，在托管的页面中强制删除此属性应防止其滥用。

3. **Privilege  Abuse.** 将其加载到沙盒属性的iframe内，可以确保用户不授予对Web托管服务的权限。某些服务（例如Google Translate）已经采用了此方法。

4. **Credential  Theft.**  针对密码管理器的一种可能的防御方法是将凭据与域路径关联，而不仅仅是域。 但是，只有少数浏览器才采用了此功能。

5. **History   Theft** History   Theft取决于网站指纹的准确性。 防止网站指纹的一种可能方法是从所有重新托管的网页中强制删除对Cookie或localStorage的调用的JS代码。 这种方法并不完美，因为已知要彻底检测混淆的JavaScript代码非常困难。更积极的方法是删除所有JavaScript代码，但这可能会严重影响网站外观或功能。

6. **Session   Hijacking   and   Injection** 因此，可以通过启用HttpOnly属性。这是使用缓解XSS会话劫持的常规措施。

## CONCLUSION
SOP是广泛被web安全研究人员关注的web安全策略，它的设计保证了域和域之间资源的隔离。作者本文的工作，发现了绕过SOP的安全漏洞，并在此基础上基于常见的web攻击进行了深入的挖掘。
在本文中作者介绍了service worker产生的安全漏洞尚未被解决（需要用户手工查看），如果有兴趣，可以在后面研究一下是否可以设计自动化工具检测service worker的安全问题。
