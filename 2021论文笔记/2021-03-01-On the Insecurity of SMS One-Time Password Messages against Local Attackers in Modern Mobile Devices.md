# On the Insecurity of SMS One-Time Password Messages against Local Attackers in Modern Mobile Devices

>作者：Zeyu Lei<sup>1</sup>, Yuhong Nan<sup>1</sup>, Yanick Fratantonios<sup>2</sup>, Antonio Bianchi<sup>1</sup>
>
> 单位：<sup>1</sup>Purdue University（普渡大学）, <sup>2</sup>EURECOM（欧洲通信学院） & Cisco Talos（思科安全智能研究与分析团队）
>
> 会议：NDSS 2021
>
> 论文链接：[On the Insecurity of SMS One-Time Password Messages against Local Attackers in Modern Mobile Devices](https://www.ndss-symposium.org/wp-content2021-03-10/ndss2021_3B-4_24212_paper.pdf)

## Abstract

**包含一次性口令（One-Time Passwords，OTP）的SMS消息**被广泛应用于身份验证中。

本文研究了“local attacks”条件下，攻击者的恶意应用程序，如何窃取到受害者设备上收到的OTP。

作者总结了Android和iOS上，应用程序**访问收到的SMS 消息中的OTP**的三类途径；并分别分析了**恶意程序通过各类途径，非法获取OTP**的具体方式。

<!-- more -->

## 1. SMS OTP 身份验证机制

SMS（Short Message Service）：短信息服务，允许用户在移动网络上，通过手机等移动设备发送文本型短信。

OTP（One-Time Password）：一次性口令。

基本验证流程：
![](/images/2021-03-10/upload_c6411f757e6f1b1959d7d107f2267c05.png)

## 2. 应用程序读取SMS中OTP的途径

作者总结了Android和iOS上，应用程序**获取SMS消息中OTP**的**三类不同途径**。

划分依据：应用程序访问OTP过程中，所需要的**用户交互类型**。

### 2.1 Access with User Interactions
第一类途径是**methods requiring per-message user interaction**，每一次OTP的获取，都需要用户的交互操作才能完成。

* **“ask-to-copy”** 机制：Android和iOS中均有

* **“One-Tap SMS”** 机制：Android中特有

![](/images/2021-03-01/upload_5d9645dc8ce9b61abbf9c89ccd7f0c5a.png)

### 2.2 Access by Requesting SMS Permissions

第二类途径是**methods requiring SMS-related permissions**，需要用户手动为该应用分配SMS相关的权限，主要包括**READ_SMS**权限和**RECEIVE_SMS**权限。

操作系统和Google Play应用市场对相关权限均有政策限制。

### 2.3 Fully-automated Access via Modern SMS APIs

第三类途径是**fully-automated methods that do not require user interaction**，该途径完全自动化，无需用户的参与，对用户透明。

这种途径主要存在于Android中，因为Android引入了三种不同的 “Modern SMS API”，以支持完全自动化的SMS OTP身份验证方式。这三种API分别是**SMS Retriever**、**SMS Token**和**SMS Token+**。

#### 2.3.1 SMS Retriever API

* **使用条件：**
  * 需要Google Play服务可用
  [*SMS Retriever API仅在具有10.2版本和更高版本Play服务的Android设备上可用*](https://developers.google.com/identity/sms-retriever/request)

* **认证流程：** 服务器端计算哈希码
  ![](/images/2021-03-01/upload_93994c8de426d0e8811c92ec24d1d116.png)
  
  其中哈希码的计算方法如下：
  ![](/images/2021-03-01/upload_555f57c24560181aa7c68be37d6b8cf0.png)
  * **concat** ：字符串拼接
  * **truncate(string, X)** ：截取string的前X位作为返回值


#### 2.3.2 SMS Token API

* **使用条件：**
  * 2017年8月Google开始提供，具有Android 8或更高版本的设备均可用

* **认证流程：** Android操作系统生成Token
  ![](/images/2021-03-01/upload_48e862a6c2a556ba92b744665e94c17b.png)
  * Token：每次createAppSpecificSmsToken()方法被调用时， SMS Token API都会生成一个随机Token（一小段文本），例如：**Eqhn SOxhzw**

#### 2.3.3 SMS Token+ API

* **使用条件：**
  * Android 10 引入

* **认证流程：**
按照官方文档的说法，SMS Token+与SMS Token基本相同，区别主要是**允许开发人员自定义SMS前缀过滤列表**。
实际的实现中：**Android操作系统生成的Token，始终等于调用该API的应用程序的哈希码，而不是随机值**。

#### 2.3.4 三个API的其他主要特点

![](/images/2021-03-01/upload_c04b5d78b9d2af601e4eb60b9bbc464c.png)


## 3. 恶意应用借助这三类途径，窃取到用户OTP的方法

### 3.1 攻击条件
* 攻击者能控制一个安装在受害者设备上的恶意应用
* 该恶意应用能通过网络发送数据


### 3.2 针对methods requiring per-message user interaction

**针对第一类途径**，作者提出攻击者可以通过欺骗用户获取OTP。
1. 恶意应用向用户询问其电话号码
2. 恶意应用联系合法应用的后端服务器，以受害者的电话号码作为号码，远程请求受害者合法帐户的SMS OTP消息。
3. 受害者将收到包含合法帐户OTP的SMS消息。
4. 恶意应用会想方设法欺骗用户，让用户将收到的OTP粘贴或输入到恶意应用的界面中。

针对该攻击方案，作者使用名为Marvel App 的在线工具，进行了用户实验。实验界面如下图所示。

![](/images/2021-03-01/upload_50f35509ad2e11f935d9068d8ec3b19e.png)

![](/images/2021-03-01/upload_3773f267242f1760e87e5cf629cea22e.png)

一旦被实验者将OTP插入恶意应用中，便认为该OTP已被盗用。实验结果如下所示。

![](/images/2021-03-01/upload_2d715ab04241ca19059dca65f68c5f6d.png)

### 3.3 针对methods requiring SMS-related permissions

**针对第二种途径**，作者提出攻击者可以**绕过操作系统限制（Bypassing OS-level Restrictions）**和**绕过应用市场限制（Bypassing Market-level Restrictions）**，来实现攻击。

#### 3.3.1 绕过操作系统限制

关于绕过操作系统限制，作者认为用户普遍缺乏对SMS相关权限的了解，并不了解允许应用读取SMS消息的后果，并进行了用户问卷调查，问卷界面及调查结果如下所示。

![](/images/2021-03-01/upload_f81264262a3d2576f05247338e177362.png)
![](/images/2021-03-01/upload_effa7a7bfadedc984b62c65bb0bbf10a.png)

#### 3.3.2 绕过应用市场限制

关于绕过应用市场限制，作者提出了两个攻击点。

其一是，当应用程序的被首次上传到应用市场（文中为Google Play）时，对SMS相关权限的审查较为严格；但当**应用的更新版本**被上传时，相关审查存在缺失。作者认为这可能是由于版本更新时，Google Play没有进行人工筛查。

其二是，可以通过获取名为BIND_NOTIFICATION_LISTENER_SERVICE的用于读取通知的权限。**所有收到的SMS消息**都会触发**包含其内容的通知**。 因此，具有读取通知权限的应用程序可以读取包含OTP的SMS消息。

### 3.4 针对fully-automated methods that do not require user interaction

#### 3.4.1 针对SMS Retriever API

针对SMS Retriever API，攻击主要利用开发人员的API误用——很多开发人员将**哈希值的计算过程**放在了**应用程序端**进行，再由应用程序端发送给服务器端，而没有按照安全标准，将该计算过程放在**服务器端**进行。
![](/images/2021-03-01/upload_aa3fccb8efbc7679aaa2b0525aaa9e6e.png)

#### 3.4.2 针对SMS Token API
![](/images/2021-03-01/upload_e817a4b2ec5a84ab7beec8cf5b5f22c4.png)

#### 3.4.3 针对SMS Token+ API

针对SMS Token+ API的攻击过程，与针对SMS Token API 的攻击过程相类似。

但是，作者通过逆向工程分析发现，实际实现中，SMS Token+ 机制下，**Android操作系统生成的Token，始终等于调用该API的应用程序的哈希码，而不是随机值**。

（因此正确的用法是，忽略API调用的返回值，直接使用哈希码）

#### 3.4.4 Modern API 的其他安全问题

* **Modern APIs’ Inbox Management**
  SMS Retriever、SMS Token、SMS Token+旨在仅将SMS OTP传递给特定的应用程序。因此，使用这些API收到的SMS消息**不应存储在SMS Inbox中**。 否则，能够获得读SMS权限的恶意应用就可以读取它们并获得OTP。
  *   SMS Retriever：始终将收到的SMS消息存储在Inbox中。
  *   SMS Token、SMS Token+：只有当设备上存在与SMS消息的Token相对应的app时，消息才不会被放入Inbox。

 * **Cryptographic Weaknesses** 
     SMS Retriever 中哈希码的计算方法如下：
    ![](/images/2021-03-01/upload_555f57c24560181aa7c68be37d6b8cf0.png)
    这将哈希算法的强度降低到66 bits。
    按照，NIST标准，SHA256返回值，不应被截断到少于224 bits。

## 4. 针对Modern API的实验结果

### 4.1 静态分析筛选

* 针对SMS Token和SMS Token+：
  * 本质上就不安全
  * 检测createAppSpecificToken()和create...WithPackageInfo()方法的调用情况
  
* 针对SMS Retriever：
  * 检测应用程序中，是否有**计算哈希码+发送哈希码到服务器**的过程：
    * 通过观察哈希码生成相关method的调用情况
    ![](/images/2021-03-01/upload_11e6e53f5efb06d61e4ffa10d42191bf.png)
    * 通过观察网络相关method的调用情况


  * 检测应用程序中，是否有**访问本地硬编码哈希码+发送哈希码到服务器**的过程
    * 作者自行计算该应用的哈希码，并进行字符串匹配
    * 通过观察网络相关method的调用情况

### 4.2 动态分析验证

作者手动与该应用的服务器进行交互，触发其身份验证过程。

### 4.3 实验结果

**数据集：**
2019年12月至2020年2月之间，从Google Play中获取到的**140,586**个应用程序

![](/images/2021-03-01/upload_be571f7e7846b824223c6c3496241c80.png)
* Suspicious：通过了手动逆向分析，确认使用了相关API
* Confirmed：作者确认了存在漏洞，且未得到修复
* Fixed：作者确认了存在漏洞，且已得到开发人员修复

Confirmed : 在 Google Play中总安装次数超过1.33亿次

Confirmed + Fixed : 在 Google Play中总安装次数超过2.3亿次