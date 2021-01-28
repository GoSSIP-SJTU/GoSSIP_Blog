# Why Eve and Mallory Still Love Android: Revisiting TLS (In)Security in Android Applications

> 作者：Marten Oltrogge，Nicolas Huaman，Sabrina Amft，Yasemin Acar，Michael Backes，Sascha Fahl
>
> 单位：CISPA Helmholtz Center for Information Security，Leibniz University Hannover
>
> 会议：USENIX Security 2021
>
> 链接：[Why Eve and Mallory Still Love Android: Revisiting TLS (In)Security in Android Applications](https://www.usenix.org/system/files/sec21summer_oltrogge.pdf)

## 简介

Android 7.0以后引入了Network Security Configure (NSC)机制使得开发者可以直接通过XML文件配置实现自定义证书验证，以缓解开发商自己实现的TLS中证书验证代码中常出现的漏洞的现象。

作者对NSC在应用中的使用情况进行了大规模分析，收集了1,335,322个Google Play上的应用，找到了99,212个应用中包括了自定义的NSC设置，但是其中超过88.87%的应用对默认的安全设置进行了降级，只有0.67%的应用实现了证书pinning。



<!-- more -->

## 背景



### NSC



 ![img](/images/2020-12-29/fig1.png)

- Cleartext Traffic Support

用于标志应用是强制使用HTTPS还是允许HTTP。Android 9之后默认是HTTPS，如果想要支持HTTP需要显式设置cleartextTrafficPermitted。也可以在manifest里设置android:usesCleartextTraffic属性（默认是true，在没有NSC的情况下生效）。

- Certificate Pinning

开发者指定`<pin-set>`。至少一个从服务器证书链发送来的证书可以匹配到注册的pin才可以建立连接。

- Custom Trust Anchors

开发者可以定义信任的CA证书和允许用户安装CA证书。自从Android 7以后，用户安装证书不能成为信任根。所以使用`user`允许用户安装证书属于对安全的降级。

- Debug Setting

允许调试时使用的本地颁发或者自签名证书。

**NSC的局限**



在Android早期版本中证书验证API仍然可以使用，这使得开发者仍然可能使用错误的证书验证代码，在某些情况下甚至覆盖NSC的设置。

### Google Play



2016-2017 Google不允许新App或者更新的App包含不安全的接口X509Trustmanager/HostnameVerifier以及方法onReceivedSslError。

2018年以后Google Play只接收目标Android版本8.0的应用或者更新，也就是说这些应用默认设置下强制用户安装证书不可信。

2019年后期，目标版本为Android 9.0，默认设置强制使用HTTPS。

## NSC使用及安全



### 数据集



作者从Google Play上爬取了Android 7.0以上的免费应用1,335,322个，其中99,212个应用有NSC设置，剔除被混淆的应用后得到数据集96,400个应用。

![](/images/2020-12-29/fig2.png)

### NSC安全性



![img](/images/2020-12-29/tab3.png)

`<base-config>`元素的设置对全部通信有效。`<domain-config>`对部分主机的通信有效，这个标签下会有下级标签`<domain>`指定设置生效的主机。表3中是对这两种设置生效的属性，以及这种属性的安全设置和不安全设置。

 <img src="/images/2020-12-29/fig3.png" style="zoom:67%;" />

### 明文流量



作者总共发现了89,686个App用了cleartextTrafficPermitted，其中88,769（98.98%）的应用设置了允许HTTP通信。只有4,804的应用没有被该元素影响。只有2,459的应用使用了这个标志强制使用HTTPS。允许在所有domain上都允许HTTP的应用是只在特定domain上允许HTTP应用的两倍，而在某些特定域上显式强制HTTPS的更多。

 <img src="/images/2020-12-29/tab4.png" alt="img" style="zoom:67%;" />

表8是应用常见的HTTPS降级的域。127.0.0.1和localhost似乎是没有安全风险的，可能用于本地的调试，或者是因为代码拷贝。

 <img src="/images/2020-12-29/tab8.png" alt="img" style="zoom:67%;" />

表5为可以升级为HTTPS的域。

![img](/images/2020-12-29/tab8.png)

#### Pinning Certificates



作者发现了663个app里使用了pinning，与2,781个domain有关，其中998个是有效的domain。

- 固定证书。作者发现了483个叶子证书，542个中间CA证书，289个根CA证书。

- 备用证书。作者发现了566个应用使用了备用证书，其中47个的备用证书无效（空字符串hash的base64，重复字符等），仅仅是为了通过开发检查。

- 证书到期。设置到期时间，到期后不会再固定。作者发现了130个应用设置了到期时间，平均到期时间是947天。无安全影响。

#### 自定义CA



作者在38,628个应用里(37,562 globally, 1,781 for domains)发现了自定义CA设置。759个应用不信任预装的CA而添加了自己的CA证书。123个应用限制了预装系统CA列表。

#### 用户安装证书



作者发现了8,606个应用 (8,001 globally, 707 for domains)重新信任了用户安装的证书。这会造成中间人攻击。

#### Debug Overrides



`<debug-overrides>`指定调试时具有的特性，如可以信任用户安装证书来进行TLS的调试。作者在318个应用程序里发现了使用`<debug-overrides>`注册自定义证书，9,904个应用允许用户应用安装证书。

作者发现了41个应用没有使用`<debug-overrides>`来进行配置自定义CA以达到调试的目的。这可能构成了安全威胁。

#### 畸形的NSC文件



1. 在`<domain>`参数里没有正确使用hostname，而是使用了URL (http://example.com/) , 资源 (@string/host), 通配符 (*.example.com) 等。

2. 把system证书的overridePins属性设置成了True。

3. 复制粘贴不安全的配置。作者发现了496个应用的配置片段（用于解决HTTP不可用的异常）可以在StackOverflow/MoPub等找到，还在1,609个应用中找到了listing5所示的片段，并且没有任何改变。

![](/images/2020-12-29/listing5.png)

#### 手工分析



作者手工分析了一些重新启用明文流量的应用。

作者随机选择了20个应用，发现其中13个使用HTTP来传输用户数据，包括广告跟踪信息，个人身份信息（设备标识符）。作者还发现了一个智能家居应用和一个学校的应用会用明文传输账号信息（用户名，密码）。

## Google Play 安全保护



Google声称对3种漏洞进行了检测：

- TrustManager不能检测非法证书（TM）

- HostnameVerifiers不能检测恶意domain和hostname（HV）

- WebViewClient.onReceivedSSLError不能处理WebView里HTTPS error（WV）

作者在加上了不安全的第三方库（LB）的检测。

<img src="/images/2020-12-29/tab6.png" style="zoom:67%;" /> 

### TrustManager实现



1. 空TrustManager
2. 非空但是不安全TrustManager
   - 只验证了过期日期：TM-R-expired, TM-R-chainexpired, TM-R-selfsigned
   - 检验了是否发送了证书链：TM-R-chain

### HostnameVerifier实现



1. 始终为真的验证
2. 验证不足，是否包含指定子串。

### WebViewClient实现



1. 没有处理Error
2. 混淆的不安全的Error处理（WV-wrapper）。

### 有漏洞的第三方库



Acra 4.2.3

JSoup 1.11.1

android-sync-http 1.4.9

### Google Play上的不安全应用



作者随机选择了15,000个Android app使用CryptoGuard进行了检测，发现了2,232 (14.8%) 个app有不安全的HostnameVerifier和5,202 (34.7%)个 app有不安全的TrustManager（不能检测这些漏洞代码的可达性），共计影响了5,511 (36.7%) 个app。

## 扩展



Towards HTTPS Everywhere on Android: We Are Not There Yet

> 作者：Andrea Possemato，Yanick Fratantonio
> 单位：IDEMIA and EURECOM
> 会议：USENIX Security 2020

## 网络库



![img](/images/2020-12-29/A_tab3.png)
作者发现三个库：HttpClientAndroid, AndroidAsync, AndroidAsyncHTTP，即使在明文流量被禁止的时候也允许HTTP。9个库在证书固定和信任锚点上遵守政策，而AndroidAsync完全不支持NSC。

## 广告库



作者收集了29个最受欢迎的广告库，其中12个都要求修改NSC。
![img](/images/2020-12-29/A_tab4.png)

明文流量。11个库（MoPub, HyprMx, HeyZap, Pollfish, AppMediation，Appodeal，AdColony,VerizonMedia,
Smaato,AerServ,DuApps）都要求对所有域明文流量。

信任锚点。Appodeal和HeyZap建议开发者将User KeyStore设置为可信。

误导向的文档。文档没有明确提及这样设置的危险，而是要求开发者复制粘贴策略。

![img](/images/2020-12-29/A_tab2.png)