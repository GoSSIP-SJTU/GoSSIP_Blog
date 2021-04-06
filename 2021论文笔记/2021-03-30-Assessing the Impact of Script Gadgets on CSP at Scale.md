# Assessing the Impact of Script Gadgets on CSP at Scale
>作者: Sebastian Roth, Michael Backes,  Ben Stock
>
>单位: CISPA
>
>会议: AsiaCCS 2020
>
>链接: [Assessing the Impact of Script Gadgets on CSP at Scale](https://dl.acm.org/doi/10.1145/3320269.3372201)

##  ABSTRACT

CSP(Content Security Policy)的目的是缓解XSS的影响，已经被所有的主流浏览器支持。但是目前面临新的威胁，即script gadgets，可以使得一些非script标记转化为可执行代码，这样可以绕过CSP。作者扩展了Lekie和Weichselbaum的工作。为了利用script gadgets，代码片段所在的库需要被加载或者主机存在于白名单中。而作者发现即使网站存在CSP，并且开发者并没有使用gadgets，攻击者可以用已知的script gadgets sideload库，结合CSP的重定向匹配逻辑，可以绕过10%的安全策略。

<!-- more -->

## 概念

### CSP:Content Security Policy

为了缓解XSS的影响。可以通过Content-Security-Policy HTTP头部来配置，可以指定加载某类资源的来源。

```
default-src 'none';
script-src 'self' www.google-analytics.com
```
CSP也禁止inline scripts和event handlers的使用。但是使用‘unsafe-inline’或者'unsafe-eval'可以解除限制。CSP白名单也支持hashes和nonces。脚本的SHA hash值和指定的吻合才执行。如果指定nonces，所有脚本(内部外部)必须带有nonce属性才会执行。

```javascript=
Content-Security-Policy: script-src 'nonce-EDNnf03nceIOfn39fn3e9h3sdfa'

<script nonce=EDNnf03nceIOfn39fn3e9h3sdfa>
  // some code
</script>
```
'strict-dynamic'允许已经被允许的脚本引入另一个脚本
```javascript=
script-src 'nonce-r@nd0m' 'strict-dynamic';default-src 'self';

<script src="/script-loader.js" nonce="r@nd0m"></script>

var s = document.createElement('script');
s.src = "https://cdn.example.com/some-script-you-need.min.js";
document.body.appendChild(s);
```
重定向会影响CSP的防御能力。

```
script-src https://redir.com https://cdn.com/benign.js

https://redir.com?target=https://cdn.com/vulnerable.js
```
### Script Gadgets

JS代码片段能被用于执行payloads。如下图

![](/images/2021-03-30/image-202103101.png)

![](/images/2021-03-30/image-202103102.png)

图2，代码片段被注入到属性中，gadgets将属性赋值给innerHTML，从而被执行。

### Open Redirects

有些重定向的实现是由URL参数来指定target。
```
$redirect_url = $_GET['redir'];
header("Location:".$redirect_url)
```
如果没有很好的验证，就可以重定向到任意目标URL。

## 威胁模型

首先作者假设网站通过CSP策略将要加载的脚本域名(包含丰富的JS库)列入白名单(script-src)，但是网站可能不一定使用。

``` javascritp=
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
```

并且假设该网站可以注入攻击，允许插入任意markup。

攻击分为四步：
* 注入payload。包含script标签，指向白名单网站上的第三方库(如：AngularJS)。一个本身不会执行的标记(button)。
* 注入的script标签会加载该脚本。
* payload的第二部分触发代码片段(gadget)

![](/images/2021-03-30/image-202103103.png)

在第一步中，除了直接加载gadget library外，还可以用重定向来加载。

## 实验

爬取top 10000网站，
* 使用Retire.JS检测网站使用的JS库和框架，例如AngularJS等。 
* 检测开放重定向(参数中是否包含目标URL((domain,path,base64)))。

在9909个网站中，检测到28个不同的库。将这些库和已知的包含有script gadgets的库取交集。

![](/images/2021-03-30/image-202103104.png)

jQury，Bootstrap只能绕过strict-dynamic，不能绕过白名单。AngularJS(947个网站)能够绕过白名单，执行payloads。
发现4902个URLs可以用于开放重定向。

![](/images/2021-03-30/image-202103105.png)

研究问题: script gadget sideloading对CSP的安全性造成多大影响？

之前的工作只是发现了例如AngularJS中的代码片段可以被用来注入，实际分析中只是匹配所使用的库是否在可利用的库中，没有实际执行攻击。

作者模拟注入来确认是否有漏洞，并且利用开放重定向来sideload库，只考虑有意义的CSP。
* 使用script-src和default-src
* 不使用*，data:或者https:
* 不包含unsafe-inline
如果白名单中只是指定了特殊的脚本，那么可以利用开放重定向来加载AngularJS库。

对于部署了安全的CSP的网站，利用puppeteer来模拟注入。

结果：
2076个部署了CSP的网站中只有248个相对安全，用29个进行实验，其中24个可以绕过。

![](/images/2021-03-30/image-202103106.png)

利用 www.gstatic.com 来加载可利用的库，可以绕过20个网站。这些网站只是使用了reCaptcha API或者Chrome应用框架。

一个实际的例子: 

Snapchat

```
script-src 'self'
 https://www.google.com/
 https://www.gstatic.com/
 https://apis.google.com/
 https://www.google-analytics.com
```
尽管使用了较为严格的安全策略，由于包含有www.gstatic.com, 攻击者可以加载https://www.gstatic.com/fsn/angular_js-bundle1.js, 从而利用AngularJS中的script gadgets来执行任意代码。

由于只有10%部署了CSP，并且只有1/4可以防御常规的脚本注入。所以作者扩展研究问题，提出假设:10000个网站都部署了安全的CSP。

首先为网站生成CSP白名单,在进行实验。结果8330个网站有3441个可以绕过CSP。1654个网站只是因为开放重定向而被攻击。

![](/images/2021-03-30/image-202103107.png)

![](/images/2021-03-30/image-202103108.png)

一个例子:reddit.com

![](/images/2021-03-30/image-202103109.png)

securepubads.g.doubleclick.net加载脚本时带有随机数或者时间戳，所以无法将完整URL加入白名单，并且存在开放重定向。

## 总结
该文章主要扩展了之前的工作，进一步思考如果网站没有使用有script gadgets的JS库，(放宽了条件限制)那么是否还可以绕过CSP，以及在真实世界其影响的大小。并且对于评估对象的缺乏，作者生成符合条件的对象，进行假设实验来进一步说明问题的严重性。