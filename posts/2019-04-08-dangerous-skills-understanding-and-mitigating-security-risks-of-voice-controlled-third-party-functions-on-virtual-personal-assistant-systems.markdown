---
layout: post
title: "Dangerous Skills: Understanding and Mitigating Security Risks of Voice-Controlled Third-Party Functions on Virtual Personal Assistant Systems"
date: 2019-04-08 12:12:30 +0800
comments: true
categories: 
---

作者：Nan Zhang  , Xianghang Mi  , Xuan Feng  , XiaoFeng Wang  , Yuan Tian  and Feng Qian 

单位：Indiana University, Bloomington； Beijing Key Laboratory of IOT Information Security Technology, Institute of Information Engineering, CAS, China； University of Virginia

出处：IEEE S&P '19

原文：[pdf](https://www-users.cs.umn.edu/~fengqian/paper/vpa_sp19.pdf)

<hr/>

## 简介

VPA（virtual personal assistant)：用户通过全语音与智能音箱进行交互获取服务，询问天气、播放音乐等，其中由第三方开发的功能叫做skill。

这篇文章研究了两个较为常见的VPA：Amazon Alexa， Google Assistant。

安全风险：涉及开放、嘈杂的声音信道没有有效的认证方式；让用户辨别交互的是正确的skill或VPA是困难的。

作者提出了两种攻击方式：

VSA (Voice Squatting Attack):口述指令的不同方式导致的攻击（口音和敬词的使用），比如“open Capital One please” -> 正常：Capital One；恶意：Capital One Please/Capital Won.

VMA (Voice Masquerading Attack)：将用户的语音交给当前skill处理而不是VPA，假装退让控制权给其他服务或skill，收集用户的私密信息。

解决措施：

skill-name scanner:利用ARPABET的发音规则，检测发音相似或者存在子集关系的skill。

detector：检测skill模仿VPA的响应、解析用户意图是否有skill切换。

贡献：

第一次研究恶意skill的安全风险，提出了两种攻击方式，提出了减轻安全风险的技术。

<!--more-->
## 背景

### VPA系统

全程语音交互，使用前用特定的唤醒语：Alexa ， Hey Google

功能：天气预报、闹钟设定、任务表、语音购物、发短信打电话、操纵手机上关联APP等

### VPA Skills和生态

skill：一系列第三方app提供各种服务

skill —— Amazon 、 action —— Google

skill可以显式或隐式的调用；隐式：will it rain tomorrow? -> Weather

**交互模型 ** :

唤醒词、触发词、skill调用名：Hey Google, talk to personal chef.

调用名可与skill名不同：Scryb -> scribe。

开发人员需要定义intents和样例发音，考虑用户与skill可能的交谈方式，例如WelcomeIntent、HelpInetent、StopIntent。

**Skill服务和VPA生态**：

![img](/images/2019-04-08/fig1.png)

用户唤醒设备，设备接收声音发送到云端，云端进行声音识别转化为文本、找到被调用的skill，将文本、时间戳、设备状态和其他的meta-data请求发送给web service。

web service将response以文本或者SSML（Speech Synthesis Markup Language）的方式发送给云端，云端处理为语音后，由设备播放给用户。

skill是不需要安装的，上传后即可通过语音调用。

## VPA 声音控制

### A. 分析VPA 声音控制

在Amazon上调用名相同或相似的skill很多，并且每次调用的时候似乎是随机选择的。Google不能有相同的但是仍有相似的。并且两种都不能处理发音的问题。

开启另一个skill时会关闭当前skill。

没有skill关闭的明显指示器，即使有一个指示灯，指示灯的显示但是不容易被用户注意到。

#### 调查研究

作者做了问卷研究以下几个问题：

•在调用sill时你会说什么？

•你有没有引用错误的skill？

•与skill交谈时是否尝试过上下文切换？

•您是否在关闭skill时遇到任何问题？

•您如何知道skill是否已经终止？

最终收到了在Amazon上105个有效的回应和Google上51个有效回应。人均有1.5个设备、每人每周平均使用5.8个skill。

![img](/images/2019-04-08/tab1.png)

85%的人使用自然语言，所以调用一个错误skill的几率更大。50%的Amazon Echo的用户和41%的Google Home用户会使用"please"。

28%的用户调用过错误的skill。

接近半数的用户会在使用skill的时候调节声音，即VPA 服务或调用另一个skill。

有30%的用户关闭skill失败过，78%的用户不会使用指示灯确认skill是否关闭。

### B. VSA(Voice Squatting Attack)

![img](/images/2019-04-08/tab2.png)

#### 威胁：

有些skill需要用户的私密信息，进行伪装可以收集。eg. Find My Phone需要电话号码；Transit Helper需要家庭地址; Daily Cutiemals需要邮箱地址。

可以进行钓鱼。例，Alexa允许运行skill在其对用户的响应中包含家庭卡，该家庭卡通过手机或web上的亚马逊配套应用程序显示，如图2进行模仿，可以使得用户拨打恶意电话进行钓鱼。

![img](/images/2019-04-08/fig2.png)

模仿一个skill，故意表现的很差使得用户对真正的应用差评。

#### 实验

作者通过实验研究了VPA对声音的识别情况。

实验准备：开发了一个将调用命令利用VPA本身的语音识别功能转为文本输出的skill。

声音命令来源：

1. 招募了20个参与者，每个人记录了400个调用命令。所有参与者能流利的使用英语，19个为native speaker。

2.  MacBook Pro使用Amazon和Google的TTS(Text-to-Speech Service)。

实验设备：Amazon Echo， Google Home Mini

实验环境：安静的房间里，一英尺的距离

随机选择了100个skill，对每一个skill，从TTS获得了10次调用名、10次调用用语，从5个参与者每人获得2条调用用语。

使用skill输出的文本中由错误识别的部分命名攻击skill，比如，如果“Capital One” 被识别成“Capitain One”，就将作者准备的skill命名为后面这个，再播放之前记录的声音命令，观察会调用哪一个skill。（即使发音错误也可以正确识别skill）。

![img](/images/2019-04-08/tab3.png)

对于额外添加词语的实验情况。

![img](/images/2019-04-08/tab4.png)

### C. VMA（Voice Masquerading Attack)

Amazon没有明显的skill结束或切换的标志。

Google会在调用的时候说“Sure, here is "+调用名和一个特殊的语音标志，不同的skill会用不同的口音和语音标志区分，结束的时候也会有特别的语音标志。但是这些语音标志和用于都可以使用SSML记录播放，这些都没有被参与者察觉过。

**假结束**:

78%的参与者依赖skill的回应（"Goodbye"或者沉默）来判断skill是否结束。

Alexa和Google Assisant支持skill在用户有段时间没有响应时发送reprompt，A 6秒，G 8秒，并且可以重复进行一次，如果用户还没有响应就自动关闭。

只要攻击者在reprompt中包含一段长达A 90s/G 120s的空白录音，那么至少可以在Alexa上持续102s运行，在Google上持续264s。

而且无论用户给了什么反应（"Goodbye")，这个时间还可以再延长。

用户触发StopIntent（"stop","off",etc）和CancelIntent("cancel"和"never mind",etc)，来结束skill是交给skill自己处理，只有“exit”会交付给VPA处理，强行关闭skill。但是有91%的用户使用“stop”，36%使用“cancel”，14%的使用“exit”。

虽然skill市场会对上传的skill进行审查，但是skill是在开发者服务器上运行的，很容易在审查后进行修改。

### D. 真实攻击

作者构建的skill提供能功能少于原本的skill，并且使用的欢迎语不一样。

![img](/images/2019-04-08/tab5.png)

在所有收集的9,582名用户的使用情况中，52名用户在使用这些skill期间尝试调用其他的skill，485名用户使用StopIntent/CancelIntent结束skill。

## 寻找voice squatting skills

作者在Amazon skill市场上爬下下来了23,758个skill，其中有19,670个是第三方skill。

Google Assistant skill只列在了它的app上，但是所有skill可以通过邮件等方式分享给其他用户，所以作者使用了AndroidViewClient自动点击分享按钮，再进行爬取，得到了1,001个skill。

【定义】CIN（Competitive Invocation Name)：与目标skill名字发音相似，或者是调用用语的变体（eg. my xxx）。

作者收集了11个调用前缀（my）和6个后缀（please)，构成目标skill的变体，再对skill名及其变体进行语音比对。

使用ARPABET因素编码（CMU做的发音字典)得到每个单词的音素。但是在所收集的9,120个单词中由1,564个不存在其中。作者又使用神经网络LSTM训练了字母-音素的模型进行补足。模型使用的训练集是Stanford GloVe数据集+收集的数据集。

使用编辑距离算法计算相似度。不同音素的替换权重不同，作者从CMU字典收集了9,181对音素替换。

定义替换权重：使用α替换β权重为WC(α,β)，删除是WC(α,none)，增加是WC(none,β)。其中 F(α)是出现音素的频率， SF(α,β)是α替换β的频率。

![img](/images/2019-04-08/gs.png)

之前TTS和参与者的实验中识别错误的平均距离分别为1.8和3.4，这里将阈值定为0-1, 0为相同。

![img](/images/2019-04-08/tab6.png)

Google上只找到4个相似发音的。

## 防范Void Masquerading

### Skill Response Checker(SRC)

SRC维护一组VPA系统使用的常用话语模板，以捕获由正在运行的技能生成的类似话语。每当发现skill的响应类似话语，触发警报并且通知VPA系统解决风险。

SRC通过对响应内容的语义分析与模板列表上的内容进行模糊匹配。作者使用BLSTM以Standford Natrual Language Inferences(SNLI)数据集加上20种常用的交互语句训练句子嵌入（sentence embedding)模型，将两个话语转化为高维向量，在计算它们的绝对余弦相似度，作为sentence relevance(SR)，超过阈值0.8进行警报。

### User Intension Classifier(UIC)

训练了一个分类器，解析用户意图，是否有切换，即用户是在与VPA交互还是Skill。例如，当用户在使用“sleep sounds”的时候，说出“play thunderstrom sounds”，可能是需要skill播放thunderstorm sounds，也可能是让VPA调用“Thunderstorm Sounds”应用。

作者选定了一组特征确定用户的交互方：

1）话语与skill之前的响应之间的SR。

2）话语与skill描述中的句子之间的前k个SR （k = 5）。

3）用户话语和描述语句之间的平均SR。

**结果**：

使用上一节的数据集，两位专家手动标记了550个对话是否为skill切换(Conhen's kappa = 0.64)。由于数据集中非skill切换太多，随机替换了一些skill文切换进去。

总共收集了1,100个切换和1,100个非切换作为基本事实。

使用5倍交叉验证的不同分类算法，精度为96.48％，召回率为95.16％，F-1 95.82％。

### 评估

**抵御原型攻击的有效性**

构建VMA攻击，作者从skill市场中选择了10种skill，使用每种skill作为用户记录了几次对话，并总共收集了61个话语。

通过将记录的对话中随机选择的话语替换为用于VPA系统的调用话语，手动制作skill切换实例（总共15个）。

将空响应或模仿VPA响应替换为每个会话的最后一句用语来构建一组伪造终止攻击（总共10个）。

系统能成功检测到所有25个上下文切换或模拟VPA实例。

**对现实世界对话的有效性**

对在训练阶段未使用的所有真实世界对话（9,582）进行分类，分类器确定了341个是切换，其中326个正确，精度为95.60%。

**性能**

在四核MacBoook Pro上延迟平均0.003ms，可忽略不计。

