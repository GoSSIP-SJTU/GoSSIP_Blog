---
layout: post
title: "DomainScouter: Understanding the Risks of Deceptive IDNs"
date: 2019-10-23 01:01:43 -0400
comments: true
categories: 
---

作者: Daiki Chiba, Ayako Akiyama Hasegawa, Takashi Koide, Yuta Sawabe, Shigeki Goto, Mitsuaki Akiyama

单位: NTT Secure Platform Laboratories, Tokyo, Japan; Waseda University, Tokyo, Japan

会议: RAID 2019

资料: [Paper](https://www.usenix.org/conference/raid2019/presentation/chiba)

<hr/>

## 1 Introduction
​	作者首先介绍了国际化域名IDNs的概念，相较于传统使用英文作为域名的情况，国际化域名指全部或部分使用特殊的文字或字母组成的互联网域名，如使用阿拉伯语、中文、斯拉夫语、泰米尔语等。但是目前，因为操作系统的核心都是由英文组成，DNS服务器的解析也是由英文代码交换，所以DNS服务器上并不支持直接的其他语言的域名解析，所有中文域名的解析都需要先转成punycode码。如我国政府网站http://中国政府.政务/（http://xn--fiqs8sirgfmh.xn--zfr164b/）。

​	在这篇论文中，作者首先介绍了视觉失真的IDN（称为欺骗性IDN）系统化，使读者了解与之相关的风险。在系统化的基础上，作者提出了一个DOMAINSCOUTER系统，用于检测欺骗性IDN并计算其分数。

​	最后，作者使用了DOMAINSCOUTER对欺骗性IDN进行了测试。

<!--more-->

## 2 Sytematization of Deceptive IDNs

 	在本文中，作者主要关注了为欺骗用户而伪造成与合法品牌相似的欺骗性IDN。作者对欺骗性IDN进行了如下两种的分类：.

1. 作者将欺骗性IDN分为**针对英文构成的合法品牌**的欺骗性IDN和**针对非英文语构成的合法品牌**的IDN。由于，在互联网成立之初仅支持ASCII码和英文字符集合，而且很多国际化知名品牌都使用的是英文域名；与此同时，许多非英语地区的本地品牌也开始使用本地语言和字符来创建域名。因此，需要对英文与非英文品牌名称区别对待。

  2. 作者提出了三种类型的欺骗性IDN：
            	1. **combo** (combosquatting, combing brand name with keywords)，指的是将品牌域名与一些其他的英文短语结合在一起的IDN。
            	2. **homo** (homograph)，指的是将某些字符替换为外观相似的字符的IDN。
            	3. **homocombo** (homograph + combosquatting)，指的是上述两个特征都符合且不能明确区别出的IDN。

​    将以上两个方面排列组合起来，论文将欺骗性IDN分为了六类。即为:

- 针对英文域名(e.g., example[.]test)的欺骗性IDN

  - combo IDNs (eng-combo; e.g., exampleログイン[.]test)
  - homo (eng-homo; e.g., êxämpl¯e[.]test)
  - homocombo (eng-homocombo; e.g., êxämpl¯eログイン[.]test)

- 针对非英文域名(e.g.,例え[.]test)的欺骗性IDN

  combo IDNs (noneng-combo; e.g., 例えログイン[.]test)

  homo (noneng-homo; e.g., ｲ列え[.]test)

  homocombo (noneng-homocombo. e.g; ｲ列えログイン[.]test)

​    由于攻击者可以自由使用一个或者多个任意单词添加在合法域名的前面或者后面，因此需要设计一个系统来掌握欺骗性IDN的性质。

## 3 DomainScouter System

​	这一节介绍了可以检测六种类型欺骗性IDNs的系统DomainScouter，如图一所示。

![](/images/2019-10-23/1.png)

​    DomainScouter分为以下五个部分：IDN提取（IDN extraction)，品牌选择（brand selection），图像生成（image generation），特征提取（feature extraction）和分数计算（score calculation）

​	DOMAINSCOUTER的输入是已经注册的IDN和选定的品牌域。DOMAINSCOUTER的输出是欺骗性IDN的分数，及其针对的目标。

### 3.1 Step 1: IDN Extraction

​	第一步涉及从域注册表数据库（domain registry databases）中提取已经存在的IDN。 由于对应于TLD（Top-Level Domain，顶级域名）的每个域注册表都已单独运行，因此作者从1,400多个TLD注册管理机构中收集注册域名，以研究世界上存在的所有IDN。

​	TLDs被分为通用顶级域名（generic top-level domains，gTLDs）和国家地区顶级域名（country code top-level domains，简称ccTLDs）。这里作者进一步将gTLD和ccTLD分类，以了解欺骗性IDN与TLD特性之间的关系。 	

​	作者将gTLD分为三种类型：**旧gTLD，新gTLD和新IDN gTLD**。

- legacy gTLD 旧版通用顶级域名，包含22TLD（例如，.aero，.asia，.biz，.cat，.com，.coop，.edu，.gov，.info，.int，.jobs，.mil，.mobi，.museum， .name，.net，.org，.post，.pro，.tel，.travel和.xxx）。

- new gTLD 新版通用顶级域名（gTLD，新gTLD由1042个非IDN的通用顶级域名（例如.top，.xyz和.loan）构成。

- 新IDN gTLD 新版IDN通用等级域名新的IDN gTLD由84个IDN TLD组成（例如。网址（.xn-ses554g）、.在线（.xn-3ds443g）和.ðóñ（.xn-p1acf）），它允许整个域名以本地语言和字符表示。

​	作者将ccTLD分为三种类型：**旧版ccTLD和新版IDN ccTLD**。

- legacy ccTLD 旧版国家地区顶级域名，旧版ccTLD由245个TLD组成（例如.cn，.jp和.uk）。
- new IDN ccTLD 新版IDN国家地区顶级域名，旧版ccTLD由42个IDN TLD组成（例如, .新加坡(.xn-yfro4i67o), .ôÇ²DG (.xn-3e0b707e), and .ðô (.xn-p1ai))）。

​	作者利用了商业WHOIS数据库截至到2018年5月，共收集到1435TLD下2.94亿个域名（包括IDN和非IDN）。如表一所示。

![1571794922077](/images/2019-10-23/2.png)

### 3.2 Step 2: Brand Selection

​	DOMAINSCOUTER的第二步是选择具有欺骗性IDN的品牌域。作者选择了英文和非英文品牌。

​	对于英文品牌，作者使用了三个主要的域名列表（Alexa，Umbrella和Majestic排名前100万的列表）来收集具有代表性的域名。一共选择了2310个域名。

​	对于非英文品牌，作者也利用了前面提到的三个主要的顶级域名列表，作者提取了非英语IDN，并删除了多余的域名，一共收集了4774个域名。

### 3.3 Step 3: Image Generation

​	DOMAINSCOUTER的第三步是从注册的IDN（第1步）和品牌域（第2步）生成图像，以用于步骤4中的视觉相似性计算。作者为注册IDN和品牌域中的IDN生成三种类型图像。

- **RAW图像** 第一种是原始图像，它是直接由每个域的字符串生成的，无需进行任何修改。 RAW用于指定欺骗性IDN（例如eng-homo和noneng-homo）与整个品牌域的非常相似的组合。
- **PSR图像** 第二种类型是从子字符串生成的公共后缀去除（PSR）图像，该子字符串从域名字符串中排除了公共后缀。例如，在PSR图像的情况下，由于.com和.co.jp在公共后缀中，因此从example [.] com和example [.] co [.] jp中都提取了example。PSR图片可以帮助区分具有不同后缀和相同目标品牌域的欺骗性IDN。
- **WS图像** 第三种是分词（Word Segmented，WS）图像。 通过将词分割算法应用于域名字符串来生成WS图像。 例如，exampleテスト[.]com分为example和テスト。作者使用polyglot算法实现了对多语言单词的分割。 WS图像是帮助检测基于联合欺骗性IDN，例如eng-combo，eng-homocombo，noneng-combo和noneng-homhocombo。

### 3.4 Step 4: Feature Extraction

​	DOMAINSCOUTER的第四步是从注册的IDN（步骤1），品牌域（步骤2）及其对应的图像（步骤3）中提取特征。 此步骤旨在设计可检测第2节中定义的六种欺骗性IDN的功能。特别是，作者使用三种类型的功能：视觉相似性，品牌特征和TLD功能。作者共总结了16个特点，如表2所示。

![1570934802870](/images/2019-10-23/3.png)

- **视觉相似性特征** 

  视觉相似性功能（表2中列出的第1-3号）旨在掌握欺骗性IDN的最大特征（即IDN的外观）。作者使用了结构相似性指标（structural similarity index，SSIM index）是一种用以衡量两张数位影像相似程度的指标。这里测试的是欺骗IDN与真是IDN分别的RAW、PSR、WS图像的最大SSIM。

- **品牌特征**

  品牌特征（表2中列出的第4-12号）旨在考虑目标品牌域的特征。作者认为更流行的域会被构建为欺骗性IDN的可能性更大。作者根据RAW，PSR和WS图像分别识别出的域名在Alexa, Umbrella, 和Majestic的排名为特征4-6，7-9和10–12号。

- **TLD特征**

  表2中的第13-16号列出的TLD特征。近年来TLD的使用已发生了巨大变化，并且具有欺骗性IDN并不总是使用与目标品牌域相同的TLD。这里的特征选择了步骤1归纳的TLD作为13号特征，基于RAW、PSR和WS的目标品牌的TLD作为14-16号特征。

### 3.5 Step 5: Score Calculation

​	在分数计算模块，作者使用了机器学习中的随机森林的算法。分为训练和测试两个阶段。训练阶段，作者使用了已经被确认为欺骗性IDN、并且已经被用于钓鱼攻击或社会工程的域名，以及一些非欺骗性IDN，并将这些域名仔细地标记为欺骗性和非欺骗性。每个IDN具有如表2所示的16个特征。在测试阶段，在训练阶段生成的模型用于计算输入IDN为欺骗性IDN的概率。作者将这些概率定义为欺骗性IDN分数。分数越高，IDN欺骗用户的可能性就越大。

​	这里使用随机森林的原因，随机森林具有很好的可解释性；由于随机森林的参数包含了决策树的数目以及每个决策树要考虑的特征，所以模型易于调整；最后，在作者初步的实验中，随机森林的方法由于其他流行的分类算法。

### 3.6 Limitation

​	DOMAINSCOUTER有两个限制。

- 它的目的不是检测各种恶意域名，而只是检测可能导致用户不当行为的欺骗性IDN。这些欺骗性IDN并不总是用于特定的恶意攻击，可能也与社会利益相关者有一定的关联。
- 在步骤二种对于非英文的品牌，只是从三大排名中获取的。对于每个国家，地区和语言，可能会存在更多的非英语品牌域名。

## 4 Evaluation

### 4.1 Comparison of Properties

​	作者与做相似研究的Liu等人[^1]和Sawabe等人[^2]做了对比，从四个角度比较了DOMAINSCOUTER和其他两个系统的特征。

- 数据集。相较于其他两个系统的研究，作者使用的系统包含了更多的已经研究过的TLD。

- 目标品牌。目标品牌检测了英文和非英文的品牌域名，数量较多。

- 欺骗性IDN。欺骗性IDN的类别较为丰富。

- 使用的方法。Liu和Sawabe等人的方法较为单一，而且这两种方法都需要手动调整SSIM索引或OCR的阈值，这往往会导致误报和漏报。而DomainScouter不仅使用了多个视觉相似性，而且使用了排名和TLD特征，并使用了机器学习的方法可以自动对SSIM索引调整阈值。

### 4.2 ComparisonofDetectionPerformance

​	作者将DOMAINSCOUTER的欺骗性IDN检测性能与先前提出的系统进行了比较。

- **对先前系统的设置** 由于Liu和Sawabe等人设计的系统不是开源的，所以作者根据两个系统的论文复现了系统。

  对于Liu等人[^1]设计的系统，系统通过计算SSIM索引并设置一个阈值来检测eng-homo IDN，原始的论文的阈值是0.95，0.95阈值导致了不可忽略的假阳性，这可能是由于原始系统和重新实现的系统在字体和图像设置上存在差异所致，所以调整阈值为0.99以减少误报。对于Sawabe等人设计的系统，作者使用了使用了Sawabe等人提供的非ASCII字符和对应的类似ASCII字符之间的映射。对于数据集，作者选择了之前论文使用的数据集。

- **对DOMAINSCOUTER的设置**  作者使用了10000个标记的IDN（包括242个欺骗性（阳性）和9,758个非欺骗性（阴性）IDN）来训练机器学习模型。作者对训练数据集进行了10倍交叉验证（CV），并获得了0.981的真实阳性率，0.998的真实阴性率，0.002的假阳性率，0.019的假阴性率和F量度 平均为0.972。

  表2的最后一列显示了所有功能的权重。 权重越高，特征对正确检测的贡献就越大。 结果表明，前三个视觉相似性特征（编号1-3）比其他特征更有效。 特别是，基于单词分割图像的视觉相似度最有助于正确检测。 其余建议的功能（第4-16）也被确认有助于检测欺骗性IDN。

- **系统的检测性质**  对于每个系统共输入4,426,317个IDN，然后结果如图2所示。Liu等人的系统能检测到1514个欺骗性IDN，Sawabe能检测到931个欺骗性IDN。而DomainScouter除了检测到共检测到1552个欺骗性IDN。DomainScouter额外检测到的欺骗性IDN主要包括作者的新目标，例如eng-combo，eng-homocombo，noneng-combo和noneng-homocombo。

  ![1570952268456](/images/2019-10-23/4.png)

## 5 Measurement Study

​	本节重点介绍DOMAINSCOUTER检测到的8,284个欺骗性IDN。 

### 5.1 Characteristics of DeceptiveIDNs

- **欺骗类型** 表4提供了检测到的欺骗性IDN的细分。 系统找到了368个engcombo，1,547eng-homo，3,697eng-homocombo，144nonengcombo和2,528个noneng-homocombo IDN。结果表明，除了eng-homo IDN之外，还有许多其他类型的欺骗性的IDN。但是并没有找到noneng-homo的欺骗性IDN。

  ![1570954126919](/images/2019-10-23/5.png)

- **目标品牌** 表5列出了在检测到的欺骗性IDN中排名最靠前的10个英文品牌及其Alexa排名。
  ![1570954126919](/images/2019-10-23/6.png)
  	检测的结果突出显示了三个主要成果。第一，如前面作者假设的一样，欺骗性IDN的主要目标是在Alex排名靠前的品牌域名。第二，排名前十的目标品牌网站都有提供用户帐户和用户登录功能，对于这个结果，可能是为了将欺骗性IDN用于网络钓鱼用以窃取用户信息。 第三、 DOMAINSCOUTER成功检测到许多本文首次定义的engcombo和eng-homocombo IDN（如，针对amazon.com的欺骗性IDN）。

  ​	表6列出了10个最具针对性的非英语品牌，以及它们的Alexa排名和英语含义。
  ![1570955565217](/images/2019-10-23/7.png)
  ​	结果证明了本文首次定义和研究的许多noneng-combo和noneng-homocombo 欺骗性IDN的存在。在排名前10位的品牌中，仅针对一个目标品牌发现了Noneng-combo IDN。作者该域名中找到了许多针对地名（例如奥地利，芭堤雅和安塔利亚）和常用词（例如运动，航班和天气）的noneng-homocombo IDN。 

- **创建日期**

  ​	作者检查了欺骗性IDN何时注册并开始使用，作者使用了WHOIS数据库（由于WHOIS数据库的局限性，这里进检测到62%的欺骗性IDN）。![1570957246587](/images/2019-10-23/8.png)

  ​	图3按欺骗类型说明了每年的欺骗性IDN数量。 结果揭示了有关欺骗性IDN的两个主要事实。 首先，欺骗性IDN的数量逐年增加。 其次，本文在2014年之后注册了许多本文中新定义的欺骗性IDN（例如eng-combo，enghomocombo，noneng-combo和noneng-homocombo）。

### 5.2 Impacts of DeceptiveI DNs

​	为了了解检测到的欺骗性IDN的影响，作者调查了随着时间的推移对检测到的欺骗性IDN的访问次数。作者使用了https://www.dnsdb.info/网站来进行检测，涵盖了从2010年6月24日到2018年9月19日的时期。表7列出了对每种欺骗性IDN的DNS查询次数总和。

![1570957952510](/images/2019-10-23/9.png)

​	**生存期**  在本文中，作者根据DNSDB数据将欺骗性IDN的生存期定义为从第一次看到DNS记录到最后一次DNS查询的时间段。使用了基于Kaplan-Meier估计器的生存分析，该估计器通常用于估计网络安全性的寿命。图4显示了生存分析的结果。

![1570961177356](/images/2019-10-23/10.png)

​	x轴是生命周期（以天为单位），而y轴则是生存概率，这意味着欺骗性IDN在经过多少天后仍然存在的概率。 从图可以看出，针对英文品牌的欺骗性IDN的特征与针对非英文品牌的欺骗性IDN的特征是不同的。 特别是，eng-combo，eng-homo和eng-homocombo IDN的生存概率要比noneng-combo和nonenghomomoombo的IDN短。

### 5.3 Brand Protection

​	本节重点介绍受其合法域名所有者或权利持有者保护的欺骗性IDN。作者调查了每个检测到的欺骗性IDN，以确定其是否受到目标品牌所有者的保护。 作者使用了WHOIS数据集来提取欺骗性IDN和目标品牌域的注册者电子邮件。如果两个电子邮件相同并且电子邮件的域部分（例如，@ example [.] com）与目标品牌域（例如，example [,] com）相同，则认为欺骗性IDN被认为是受保护的。

​	通过上述识别过程，作者发现，只有3.8％（= 316 / 8284）的欺骗性IDN受其目标品牌所有者的保护。  表8列出了十大受保护品牌； 它包含Alexa等级，受保护的欺骗性IDN的数量，所有检测到的欺骗性IDN的数量以及保护率。

![1570962283513](/images/2019-10-23/11.png)

​	表中可以得到两个结论。第一，没有一个品牌域名可以保护自己免受所有相应的检测到的欺骗性IDN的侵害。 第二，只有少数提供Internet安全服务（例如Cloudare和Symantec）的公司比其他公司更能保护自己的域名免受欺骗性IDN的侵害。

## 6 User Study

​	作者在Amazon Mechanical Turk（MTurk）进行了两项单独的在线调查：“用户研究1”用于调查用户对IDN的了解；“用户研究2”用于检查欺骗性IDN对用户的欺骗程度。

### 6.1 User Study 1

​	调查一旨在向参与者询问其人口统计信息和IDN知识。 该调查包括12个封闭式问题。 使用域名来询问参与者有关字符的信息。 例如：一道选择题是这样的，问题是：IDN可以包含除标点符号以外的所有这些字符，具有以下选项：英语（大写），英语（小写），数字，连字符，标点，西里尔字母，希腊文，中文，日文和韩文。

​	结果如表9所示。一小部分参与者（只有5.5％（= 20/364））对IDN足够了解，只有11.3％（= 41/364）的参与者了解关于IDN的知识。令人惊讶的是，只有13.5％的计算机工程师或IT专业人员正确回答了该问题。

​	![1570963329838](/images/2019-10-23/12.png)

### 6.2 User Study 2

​	调查二检查利用欺骗性IDN进行的攻击，欺骗到用户的程度。

​	该调查由18个封闭式问题组成，涉及用户的人口统计信息和对欺骗性IDN的视觉感知。 为了衡量有多少用户不了解伪装成流行在线服务域的欺骗性IDN，作者为七个流行品牌（在线服务）准备了70个实际的欺骗性IDN，这些品牌分别是Google，YouTube，Facebook，Amazon，Twitter，Instagram和PayPal。作者为每个目标品牌准备了五个得分高的欺骗性IDN（得分为1.0）和五个得分低的欺骗性IDN（得分为0.06至0.56）。

​    在回答了几个虚拟问题之后，作者向参与者提出了一个欺骗性问题，询问“您是否曾经访问过[SERVICE] .com？”作为封闭式问题，可以用“是”或“否”回答。 在这个问题中，[SERVICE] .com实际上已被具有欺骗性的IDN所取代。 例如，将使用êxample[.] test代替了example [.] test。 对每个参与者随机显示的欺骗性IDN及其顺序。 作者将此次攻击的潜在受害者定义为在上一个问题中每月使用服务超过一次的人中，即回答“是”的人。作者假设回答“否”的参与者识别出欺骗性的IDN。

​	调查的结果如表10所示。作者定义不可知率$Insensible Rate = v/p$，其中p是回答他们每月使用某个品牌服务一次的参与者数量，v是回答他们访问了欺骗性IDN伪装该品牌服务的潜在受害者的数量 。大多数参与者并没有真正注意到得分高的欺骗性IDN。 得分高的IDN的误判率为0.92（= 1,153 / 1,242）。即使一些欺骗性IDN的分数不高，许多参与者也没有注意到欺骗性的IDN。得分较低的IDN的误诊率总计为0.79（= 994 / 1,253）。

​	![1570965785696](/images/2019-10-23/13.png)

​	作者还计算了欺骗性IDN和Insensible Rate的相关性，结果如表11所示。结果表明，所提出的系统可以成功地计算出合理的分数，以反映用户被欺骗性IDN攻击的可能性。

![1570970115901](/images/2019-10-23/14.png)

## 7 Discussion

​	作者就如何减少欺骗性IDN的传播，分别向客户应用程序，域注册商/注册机构，域所有者和证书颁发机构（CA）提出了相关的建议。

### 7.1 Client Application

​	客户端应用程序（例如网络浏览器和其他能够显示URLs的应用）可以通过检测欺骗性IDNs并阻止用户访问IDNs来防止用户被欺骗性IDNs欺骗。但是，客户端应用程序中的缓解措施只能阻止用户访问欺骗性IDN，而无法从源头上解决欺骗性IDN的产生。

### 7.2 Registrar and Registry

​	主要针对TLD注册管理机构实施IDN的指南规定，不允许将视觉混淆字符共存于单个IDN标签中，由于本次检测中这种类型的欺骗性IDN占大多数，所以如果注册管理机构严格遵守禁止，则可以避免大约一半的欺骗性IDN。

​	尽管商标信息交换机构（TMCH）致力于保护域名，但检测欺骗性的IDN超出了其技术范围。 技术问题是域名被视为与TMCH记录的字符串完全匹配，此方法会导致假阴性。

### 7.3 Domain Owner

​	品牌保护是品牌域名持有人抵制他人侵犯其权利的重要途径。那些拥有知名域名（或商标）的人应该努力保护自己的品牌，他们可以主动注册与自己品牌相似的其他域名，以防止其他方滥用注册。

### 7.4 Certificate Authority

​	许多CA已向被抢注的域名颁发了证书，包括欺骗性IDN。作者建议CA应与域名注册商共同遵循的品牌保护政策。 如果所有CA主动共享类似于TMCH的商标信息，则他们就不会向欺骗性ICN颁发证书。此外，如果CA收到了品牌域名持有者的要求，则可以撤销为违反商标的域名颁发的证书。于此同时，域名所有者能够在证书透明性的日志服务器中浏览被抢注域名的证书的相关信息。



[^1]: Liu B, Lu C, Li Z, et al. A Reexamination of Internationalized Domain Names: The Good, the Bad and the Ugly[C]//DSN. 2018: 654-665.
[^2]: Sawabe Y, Chiba D, Akiyama M, et al. Detecting Homograph IDNs Using OCR[J]. Proceedings of the Asia-Pacific Advanced Network, 46: 56-64.
