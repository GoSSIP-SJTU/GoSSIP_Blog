# Cardpliance: PCI DSS Compliance of Android Applications

> 作者：Samin Yaseer Mahmud\*, Akhil Acharya\*, Benjamin Andow†, William Enck\*, and Bradley Reaves\*
>
> 单位：*North Carolina State University, †IBM T.J. Watson Research Center
>
> 出处：USENIX 2020
>
> 原文：https://www.usenix.org/conference/usenixsecurity20/presentation/mahmud



## Abstract

智能手机及其应用已经自然地成为金融交易技术的重要组成部分，但是，以往的研究在很大程度上忽略了要求用户输入信用卡号的应用程序，此类应用对安全性特别敏感，并且受支付卡行业数据安全标准（PCI DSS）的约束。

在本文中，作者设计了一个名为Cardpliance的工具，该工具将图形用户界面的语义与静态程序分析相结合，以捕获PCI DSS的相关要求。作者使用Cardpliance研究了358个来自Google Play的流行应用程序，这些应用程序要求用户输入信用卡号。总体而言，发现了358个应用程序中有1.67％不符合PCI DSS，并且存在漏洞，其中包括存储信用卡号和卡验证码的方式不正确。这些发现大致反映了流行Android应用程序对PCI DSS的遵守状态。

<!-- more -->

## 1.  Introduction

移动操作系统上的集中支付平台（例如Google Pay和Apple Pay）提供了安全可靠的方式来管理付款信息，但是，最近的研究表明，应用程序要求用户输入信用卡信息的情况仍然存在。

研究问题：移动应用程序是否会误处理付款信息？ 

技术挑战：

（1）哪些PCI DSS要求适用于移动应用程序？ PCI DSS v3.2.1（2018年5月）有139页，适用于多种支付系统。

（2）这些需求如何转化为静态程序分析任务？ 分析应避免 false negatives，同时最大程度地减少false positives。 

（3）如何以编程方式识别信用卡信息的使用？ 区分信用卡文本值需要了解用户界面中小部件的语义。

本文的贡献：

* 将针对移动应用程序的PCI DSS要求编码为静态程序分析规则。
* 研究了358个已知可提示用户输入信用卡信息的应用程序。
* 提出了最佳实践供开发人员处理付款信息。

作者称Cardpliance是第一个无需事先知道哪些应用程序可以处理信用卡信息即可识别违规情况的系统。



## 2.  PCI Data Security Standard

在2000年初，主要的信用卡公司面临着因广泛使用在线金融交易而导致的支付欺诈危机。 因此，支付卡行业（PCI）发布了其数据安全标准PCI DSS，规范了所有与PCI成员开展业务的金融系统，定义了安全措施。作者发现PCI DSS区分了持卡人数据（CHD）和敏感帐户数据（SAD），数据分类对软件处理的影响如图所示。

![](/images/2020-12-04/table1.png)

作者识别了以下六个PCI DSS相关要求：

**要求1（限制CHD的存储和保留时间）**：

应将信用卡号和其他CHD值写入持久性存储时的情况最小化。请勿将CHD写入共享存储位置（例如Android中的SD卡）。

**要求2（限制SAD存储）：**

SAD值决不应写入永久性存储。数据源（例如传入的事务数据、日志、历史文件、跟踪文件、数据库方案和数据库内容）不应包含SAD。不要在调试日志和故障转储中包含SAD。

**要求3（显示时遮挡PAN）**：

在用户输入信用卡号之后，应用程序应该在显示前将其屏蔽。

**要求4（存储时保护PAN）**：

补充了要求1，如果信用卡号（PAN）被写入，则需某种保护。

**要求5（使用安全通信**）：

所有网络连接都应使用TLS / SSL，不应删除服务器身份校验。

**要求6（通过消息传递技术安全地传输PAN）**：

不应将信用卡号（PAN）传递给发送SMS消息的API。 Android允许使用基于Intent消息的组件间通信（ICC）与其他消息应用共享数据，此类消息应受到保护。



## 3. Overview

作者尽可能地利用了现有的开源项目：UiRef（推断文本输入的语义）；Amandroid（执行静态数据流分析）；MalloDroid（识别SSL漏洞）；StringDroid （识别用于建立网络连接的URL字符串）。

Cardpliance识别移动应用中违反PCI DSS的方法：

（1）确定哪些应用程序要求用户输入信用卡信息：使用两阶段应用程序过滤器，首先在应用程序使用的字符串中搜索关键字，然后利用UiRef确认应用程序要求用户输入信用卡信息。

（2）数据依赖图（DDG）提取：首先使用UiRef把与信用卡信息相关的UI输入小部件标记出来，其次增强Amandroid处理OnClickListener回调的方式。

（3）六个PCI DSS测试捕获相关要求：定义了用于进行Android污点分析的source和sink。

作者偏向于false positives而不是false negatives，并通过人工检查消除false positives。 

![](/images/2020-12-04/fig1.png)

## 4. Cardpliance

### 4.1 DDG Extraction

**Amandroid中的关键概念**：

* 数据流分析。计算控制流程图中每个指令的指向信息，最后创建整个应用程序的DDG。

* DDG执行污点分析。Amandroid在DDG中标记污点source和sink，计算它们之间所有路径的集合，把路径存储在污点分析结果TAR中。

  

**配置Amandroid进行分析**：

* 使用Amandroid的`ExplicitValueFinder.findExplicitLiteralForArgs（）`方法来确定传递给source的整数值。然后使用UiRef标识的信用卡信息小部件的资源ID来确定流到每个sink的信息类型。

* Cardpliance实现了自定义source和sink管理器，将污点source细化为`Activity.findViewById(int)`指令。
* Cardpliance的一项测试使用`View.setText()`作为sink，因此优化了自定义的source和sink管理器。 
* 修改Amandroid的控制流分析，以正确跟踪`OnClickListener`回调中获取的`View`对象的使用。



### 4.2 PCI DSS Tests

Cardpliance使用Amandroid的污点分析结果（TAR）来识别潜在的PCI DSS违规行为。

#### 4.2.1 Analysis Approach

DDG是有向无环图$(V, E)$，其中顶点集$V$是程序指令，边集$E$表示顶点$(v_i,v_j)$之间的def-use依赖。

如果存在一系列边$(v_s,v_{s+1}), (v_{s+1},v_{s+2}), ...(v_{k-1},v_k)$，那么$v_s$与$v_k$之间存在一条路径（表示为$v_s \leadsto v_k$），将从$v_s$到$v_k$的特定路径$p$表示为$v_s \stackrel{p}{\leadsto}v_k$。

每种PCI DSS测试是根据调用三组方法的指令定义的：source methods（$S$），sink methods（$K$）和required methods（$R$）。

对于$V$中的$v_s,v_k,v_r$，违反项的路径如下：

![](/images/2020-12-04/formula.png)

#### 4.2.2 Test Implementation

**测试T1（存储CHD）**：确定何时将CHD写入持久性存储。作者将文本输入源简化为信用卡号（PAN），使用到了Amandroid的`ExplicitValueFinder`。

**测试T2（存储SAD**）：用户输入的唯一SAD是写在物理卡上的三位数或四位数的CVC代码，因此仅考虑传递了与CVC相关字段的资源ID的`Active.findViewById(int)`源。 

**测试T3（遮蔽信用卡号）**：除了在用户中输入时显示完整信用卡号（PAN），其余都应将其遮蔽。作者定义了一组PAN掩码方法（PMM），代表应用程序开发人员可能掩盖信用卡号的方式。另外，T3还将网络输入作为源，作者使用了启发式方法来检测哪个输入是信用卡号。

**测试T4（存储未混淆的信用卡号）**：包含一组必需的混淆方法（OM），包括对Java中常见的加密和消息摘要函数的调用。对于`Cipher.doFinal()`方法，验证是否使用`ENCRYPT_MODE`初始化了`Cipher`对象。

**测试T5（不安全传输）**：应用程序有两种不正确使用TLS / SSL的方式：

（1）通过HTTP URL发送数据：使用`OutputStreamWriter.write()`和`OutputStream.write()`作为sink，找到创建输出流对象的`URLConnection`对象，然后使用Amandroid的`ExplicitValueFinder`来确定传递给URL初始化方法（`URL.init（String）`）的参数，如果使用了HTTP URL，则会引发警报。

（2）通过HTTPS URL发送数据时使证书验证无效：利用了Amandroid API误用模块中的`COMMUNICATION_LEAKAGE`配置选项，查找`SSLSocketFactory`和使用`ALLOW_All_HOSTNAME_VERIFIER`标志的`TrustManager`的不安全实现。

**测试T6（共享无混淆的信用卡号）**：识别SMS利用了Android API `SmsManager.sendTextMessage()`。识别Android的组件间通信（ICC）利用`Intent.putExtra()`。

<img src="/images/2020-12-04/table2.png" style="zoom:50%;" />





## 5. PCI DSS Compliance Study

### 5.1 Dataset Characteristics

作者于2019年5月下载了Google Play的35个类别中的前500个免费应用程序（17,500个），利用UiRef筛选并确定了442个要求输入付款信息的应用程序。最后Cardpliance成功地分析了数据集中的80.99％（358/442）应用程序，涵盖了32个类别，其中大多数来自FOOD_AND_DRINK（51），SHOPPING（43），FINANCE（39）和MAPS_AND_NAVIGATION（37），平均下载量为125万次，平均评分为3.8星（满分5星）。

实验环境：VMware ESXi 6.4，Ubuntu 18.04，Intel®Xeon®Gold 6130 2.10GHz，320 GB RAM，28 physical cores.

Cardpliance配置：并行15个应用程序，每个应用程序组件设置60分钟的超时时间。



**发现1：Google Play上流行的免费Android应用程序中至少有2.5％直接要求付款信息。** 

**发现2：Cardpliance可以分析平均运行时间为334分钟和中值运行时间为179分钟的应用程序。**

![](/images/2020-12-04/fig2.png)


### 5.2 Validation Methodology

本文的一名学生作者进行了手动代码验证，他在Java编程和Android应用程序开发方面拥有6年以上的学术和行业经验。 对于Cardpliance标记的每个候选应用程序，首先使用JEB反编译器对应用程序进行反编译以获得源代码，然后对标记为潜在的PCI DSS违规的情况进行验证。



### 5.3 Compliance: The Good

**发现3：358个应用程序中约有98.32％通过了Cardpliance的PCI DSS一致性测试。**

**发现4：应用程序正确使用HTTPS而不是HTTP来传输付款信息。** 

**发现5：通过SSL连接发送付款信息时，应用程序正确执行了主机名和证书验证。**

**发现6：应用程序并未通过SMS或与其他应用程序通过ICC通道共享付款信息。**



### 5.4 Non-Compliance: The Bad and the Ugly

在对Cardpliance标记为可能违反PCI DSS的40个应用程序进行验证之后，作者发现有6个应用程序不符合PCI DSS要求。

![](/images/2020-12-04/table3.png)

**发现7：存在超过150万次下载的应用程序不符合PCI DSS法规。**

**发现8：应用程序存储的信用卡号没有散列或加密数据**。

![](/images/2020-12-04/fig3.png)

**发现9：应用程序保留卡验证码（CVC）**。

 “收费公路”（`com.seta.tollroaddroid.app`）将付款请求与CVC输出到设备日志。

餐厅Ben’s Soft Pretzels（`com.rt7mobilereward.app.benspretze-1`）将CVC写入设备日志。

支付通行费的FastToll Illinois（`com.pragmistic.fasttoll`）将CVC保留在shared preferences中。

**发现10：应用程序在用户界面中显示时不会掩盖信用卡号**。

ConnectNetwork by GTL（`net.gtl.mobile_app`）将用户的信用卡号作为一个UI小部件的输入，然后在另一个UI小部件中完全显示它以进行验证。



### 5.5 Case Studies

**案例研究1：信用卡阅读器应用程序错误地处理了成千上万个客户的信用卡号：**

信用卡读卡器Credit Card Reader（`com.ics.creditcardreader`）

* 保留信用卡号，而没有进行哈希处理或加密。
* UI中的EditText小部件获取用户的信用卡号，并将其直接登录到LogCat。 它暴露了的客户的信用卡号，还带来了欺诈风险。例如，如果客户获得了对该设备的物理访问权，则他们可以以明文形式下载客户的所有信用卡号。

![](/images/2020-12-04/list1.png)

**案例研究2：用于在餐厅连锁店下订单的应用程序将信用卡号码与CVC一起保留为明文**：

* 将信用卡号存储到`SharedPreferences`之前试图对其进行加密。但是，用于存储密文信用卡号的key-value对中的key是常量字符串与用户的信用卡号和用户名的串联。因此，信用卡号仍以明文形式保留在磁盘上。

* 使用用户名和信用卡号为随机数生成器提供种子，以生成密钥。该加密密钥也被写入日志和`SharedPreferences`中。

* 当用户单击“付款”按钮时，会同时记录信用卡号和CVC。如果在单击按钮时用户输入的字段为空，则还将记录剩余的付款信息（例如，检验日期，姓名，地址和邮政编码）。

* 在`CreditCardSaved2Page` 中，将明文的信用卡号和CVC代码保存到`SharedPreferences`中。如果用户返回该页面，则信用卡号和CVC都将从`SharedPreferences`中提取，并重新填充到文本字段中。

![](/images/2020-12-04/list2.png)

### 5.6 Disclosure of Findings

Cardpliance在Google Play的6个应用程序中确定了15个违反PCI DSS的行为，作者尝试通过Google Play中的电子邮件与开发人员联系。 披露后75天，只有一位开发人员响应了信息，作者在准备论文的camera-ready版时，尚未在Google Play中看到该应用程序的更新版本。

### 5.7 Threats to validity

**False Negatives**：

（1）不完整的关键字列表；（2）Amandroid尚不健全；（3）差劲的证书验证

**False Positives**：

（1）UiRef确定UI输入语义；

（2）在`findViewById(context，id)`源传播到了使用上下文对象的不相关代码；

（3）T5中对上下文不敏感的易受攻击SSL库的标识；

（4）T3测试掩码的轻量级启发式方法；

（5）T6假定隐式`Intent`仅用于应用程序之间的ICC。



## 6. Recommendations for Developers

1. **将付款处理的责任委托给已建立的第三方付款提供商。** 
2. **不要将CVC写入持久性存储或日志文件。** 
3. **避免将信用卡号写入持久性存储或日志文件中**。 
4. **在本地存储之前，请使用随机生成的安全密钥对信用卡号进行加密**。如果信用卡号必须保存在本地，则应使用由Android的密钥库管理的密钥对其进行加密。 应使用随机生成的密钥（例如不带硬编码种子的`SecureRandom`类），并遵循PCI DSS建议的密钥长度，并使用已建立的密码库（例如`javax.crypto`）。
5. **通过网络传输时，请始终通过安全连接发送付款信息。**
6. **切勿在应用程序代码中修改`SSLSocketFactory`或`TrustManager`**。 
7. **在显示信用卡号之前，请务必先遮盖它**。 只显示前六位和后四位。
8. **在Android组件之间共享付款信息时，仅使用显式寻址的`Intent`消息**。 隐式`Intent`可能会导致其他应用的无意访问。



