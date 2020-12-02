---
layout: post
title: "Security Analysis of Unified Payments Interface and Payment Apps in India"
date: 2020-03-13 17:44:22 +0800
comments: true
categories: 
---

作者：Renuka Kumar[1,2], Sreesh Kishore[2] , Hao Lu[1] , and Atul Prakash[1]

单位：[1]University of Michigan, [2]Amrita Vishwa Vidyapeetham

会议：USENIX SEC 2020

链接：[pdf](https://www.usenix.org/system/files/sec20summer_kumar_prepub.pdf)

## 简介

自2016年以来印度大力推进移动支付，2018年度有超过$50billion的交易通过移动支付完成。为了实现移动支付，印度的银联NPCI(National Payments Corporation of India)引入了UPI(Unified Payments Interface)，可以完成不同银行不同账户的即时转账，即移动支付类的App通过UPI可以完成从银行A的账户到银行B的账户直接转账的操作。UPI没有人研究，并且它是闭源的。作者对未发布的UPI1.0协议进行首个安全分析。

<!--more-->

![img](/images/2020-03-13/fig1.png)

目前有88个UPI App，支持超过140家银行。Google Pay (Tez), PhonePe, Paytm, BHIM四款App占了88%的市场份额。BHIM是NPCI的官方应用，88个UPI App中大多数App是BHIM的变体，即某个银行定制版。

![img](/images/2020-03-13/tab1.png)

由于Android占了超过90%的印度手机市场份额，这里只研究App的Android版本。

作者的贡献：

1. 对未发布的UPI1.0协议进行首个安全分析。
2. 展示了在无法访问UPI服务器的情况下分析应用层协议的方法。使用BHIM进行初步分析，并在其他应用确认分析结果。
3. 展示了攻击者进行远程攻击的方法。
4. 向印度CERT和美国CERT报告了这些缺陷，获得了多个CVE，部分问题在UPI2.0中得到解决。
5. 使用BHIM，Google Pay，Amazon Pay和PayTM（印度四个最受好评的UPI 2.0应用）对UPI 2.0进行了持续分析，得出了早期发现。调查结果表明仍存在一些漏洞。

## 背景

### UPI App的用户注册

用户使用**手机号码**注册：a)查找银行卡时可作为digital identity的代理，2）接受 SMS OTP，3）交易提醒。

用户可以有多个UPI App，每个App上的账号都会使用唯一且不同的**UPI ID**进行表示。一个UPI App的账户中可以添加多个银行卡。

用户注册后，UPI服务器会向App发送银行卡列表。用户在第一次使用银行卡时需要输入**银行卡号后6位和有效期**来验证银行卡。第一次交易之前，设置一个UPI PIN，用于之后交易的身份验证。

Alice向Bob转账：首先Alice输入密码登录账户；Alice向Bob询问其UPI ID（一般是手机号）；Alice选择他添加过的一个银行卡，生成一个对Bob的交易，输入UPI PIN，完成交易。

### 用户注册的UPI规范

在用户注册时，App读取手机号或者用户输入手机号后，App立即自动向UPI服务器发送加密的SMS，确认手机号码和设备的设备的绑定关系。手机号和设备信息（Device ID，App ID，IMEI number等标识）被称为设备的**指纹**，是UPI 规范认证的第一步。

账户密码不在UPI规范中，由App厂商认证，即用户有双重认证：UPI服务器认证设备指纹和UPI PIN，App服务器认证用户密码。

### 威胁模型

攻击者发布一个具有INTRTNET和REVECIVE_SMS权限的App，被用户下载安装在手机上。

部分攻击还需要READ_PHONE_STATE（用于读取手机号码）或者accessibility权限（用于监控用户输入）。

## 安全分析

### BHIM 用户注册的协议

*握手协议流程*：

1. Alice打开App，App申请权限；获得相应的权限后，将设备信息以HTTPS发送给UPI服务器。

2. UPI服务器给Alice发送13位的注册Token，等待Alice返回SMS。

3. BHIM app用SMS发送注册token给UPI服务器。

4. UPI服务器收到SMS后绑定账号和手机，返回一个确认信息给BHIM。

5. BHIM发送一个HTTPS请求向UPI询问绑定状态。

6. UPI响应状态，包括Alice的customer ID，注册token。

   至此，UPI服务器完成用户和设备的绑定。

7. BHIM让Alice设置密码，将密码的SHA-256和手机号一起以HTTPS POST发送给UPI服务器。

8. UPI服务器返回一个登录token，账户新建成功。

9. BHIM展示UPI支持的银行列表，Alice选择后BHIM发送银行ID给UPI服务器。

10. UPI服务器发送Alice的银行账户信息给BHIM。

![img](/images/2020-03-13/fig3.png)

*其他流程1*

​	如果第三步后UPI没有收到BHIM的SMS，可以通过4a中流程替代。Alice输入手机号后，BHIM使用HTTPS消息发送手机号和注册token给UPI服务器。UPI服务器发送OTP给Alice，Alice在app中输入进行绑定。

![img](/images/2020-03-13/fig4.png)



*其他流程2：*

​	已注册用户在更换手机时也需要重新进行设备绑定，在第6步中accountExists为true。更换手机只用输入密码，不需要重新添加银行账号，登陆后由UPI服务器直接提供。

#### 攻击1：获取手机号进行未授权注册

Mally app：由攻击者Eve发布的，有INTERNET,RECEIVE_SMS和READ_PHONE_STATE（读取手机号）权限。安装于受害者手机上。

Eve会建立一个 command and control (C&C) 服务器。并且他手机上的BHIM为重打包版本，去掉了客户端的安全检查。

1. Mally在Alice手机上安装后，向Eve的C&C服务器发送Alice的手机号。
2. Eve手机上开启飞行模式后，连接WIFI注册BHIM账户。BHIM的短信会投递失败，通过替换流程1的HTTPS信息完成绑定，UPI服务器会发送OTP给Alice。
3. Mally读取短信后发送C&C服务器。
4. Eve发送OTP。一般BHIM会检查UPI服务器是否是已知的，但是Eve手机上是去掉检查的版本。
5. 如果UPI服务器返回的accountExists是false说明是新用户，设置密码进行注册。

#### 攻击1‘：克服BHIM对已存在用户的密码检查

在攻击1中第5步，如果是已存在用户，存在两种可能的攻击方式：

1. Mally等待Alice登录BHIM，在BHIM输入密码的登录界面绘制叠加层（利用CVE- 2017-0752 ）如图4c，获取到密码。

2. Mally请求并使用accessibility权限，可以观察用户交互并拦截密码。

此时攻击者可以重置账号密码。BHIM的设计重置密码后需要用户输入银行账号，如其他流程2中所述，UPI服务器会返回用户的银行帐号。Eve可以使用银行帐号来重置Alice的BHIM密码。

攻击1和1’的影响：无法发起交易，但是可以获取一些Alice的信息，比如她在哪些银行办理了银行卡，为攻击2和攻击3提供条件。

#### 攻击2：获取手机号和部分卡号后进行未授权交易

在退房缴费/买单等用银行卡进行付钱的场所，可能会泄露卡面上的卡号和有效期，这种情况下一般还伴随手机号的泄露（用于积分）。在攻击1成功的前提下，就可以在账户中加入该银行卡，进行交易。

影响：这个攻击影响的范围较攻击1更小，但是在Eve收集的信息中同时满足安装了Mally和泄露了银行卡，就可以设置UPI PIN，直接进行消费。

#### 攻击3：无卡号的未授权交易

这个攻击是在攻击1‘成功的条件下进行，可以使用同样的overlay方法或者accessibility权限获取UPI PIN码。但是，如果想要重置PIN码就需要卡号信息，

#### 没有READ_PHONE_STATE权限下

即Mally无法直接获取手机号码，给定一个目标手机号的集合C：

1. C&C服务器向C中所有手机号发送给具有特定内容的短信[接收方手机号，“SMS TEST"]
2. Mally读取接收到的短信，通过"SMS TEST"标识，获取到手机号。

### 其他UPI 1.0 App

作者测试研究期间最受欢迎的三个应用程序：PhonePe，Ola Wallet和Samsung Pay。

攻击1：PhonePe提供了注册时输入自己手机号的选项。Eve设置密码时需要OTP也通过Mally转发完成。攻击2也可以完成。

对于攻击1‘来说，PhonePe重置密码只需要OTP；Ola Money还需要注册时的一个秘密信息，作者说可以进行拦截，但是没有具体说怎么拦截。特别地，Eve一旦登录Alice的账号，对于PhonePe应用Alice会被弹出，而Ola Money允许多登录，原本Alice手机上的账号不会有任何提示。

Samsung Pay (SPay)使用了TEE。使用SPay必须具有三星账号，并且需要设置指纹或者SPay PIN。SamsungPay不与UPI集成； 相反，它与两个UPI App集成在一起：Paytm和MobiKwik。 因此，用户可以选择SamsungPay附带的两个应用程序之一（它们也可以在Google Play上单独下载）。 由于Paytm和MobiKwik应用服务器均未与KNOX集成，因此在用户注册时，它们无法使用KNOX的基于硬件的安全功能对设备进行硬绑定。 用户的指纹或SPay PIN用于通过设备验证用户身份； 支付应用程序服务器或UPI服务器均未将其用于用户注册。 作者使用MobiKwik测试SamsungPay。 Mobikiwk的工作流程与Ola Money相同，只不过其密码重置工作流程使用密码和OTP。 这使得SamsungPay容易受到与第三方UPI应用程序集成而导致的攻击。

### CVEs

BHIM: CVE-2017-9818,CVE-2017-9819,CVE-2017-9820,CVE-2017-9821

其他App：CVE-2018-15660, CVE-2018-15661,CVE-2018-17400, CVE-2018-17401, CVE-2018- 17402,CVE-2018-17403

三星：CVE-2018-17083

### UPI2.0 协议分析

UPI2.0去掉了其他流程1，即解决部分安全问题，但是其他漏洞仍然存在。对于UPI2.0，作者分析了BHIM，Google Pay，Paytm和Amazon Pay四个流行应用。

在GPay上，作者可以实现类似于在第三方UPI 1.0应用中对Attack 1和Attack 1‘所做的操作。GPay使用OAuth2对Gmail服务器进行了身份验证。因此，Eve可以按以下方式设置GPay帐户。

1. Eve可以在手机上使用自己的Gmail ID，并在登录GPay时输入Alice的手机号码。 Google会向Alice的手机号码发送一次OTP，Mally进行转发。
2. GPay需要将包含Alice的设备注册Token的SMS消息从Alice的手机发送回UPI服务器。没有其他流程1的情况下，Eve可以进行SMS欺骗，但是作者实验没有成功。另外，Mally可以请求SEND_SMS许可并通过Alice的手机发送短信。

以下是Paytm在握手期间收到的银行帐户信息的摘要。作者将部分信息掩盖起来以保护隐私。与以前一样，UPI会发回银行帐户详细信息，而无需用户提供与银行共享的任何凭据。Amazon Pay上也一样。 Amazon Pay使用Amazon凭证和用户的Amazon帐户中设置的默认手机号。要创建个人资料，Eve可以在她的Amazon凭证中设置Alice的手机号码。

![img](/images/2020-03-13/fig.png)

## 经验教训

1. UPI协议在获取用户手机号但没有银行相关凭证下在握手协议中会显示用户银行账户信息。

2. 设备的硬绑定作为第一重验证仅依赖于易于从设备中手机的信息，不使用任何机密。

3. 弱设备绑定机制使得用户可以讲设备与另一手机号绑定。

4. 设置UPI PIN的机密信息是银行卡信息，这不是秘密。

5. 现有用户转移到新设备时UPI不需要用户提供任何银行凭证以获取授权。

6. 在第三方应用上，密码由第三方管理，容易被绕开。

7. 从任何第三方应用程序的默认工作流程中泄露的银行帐号足以在另一个应用程序（例如BHIM）上重置用户的密码。

   