---
layout: post
title: "TextExerciser: Feedback-driven Text Input Exercising for Android Applications"
date: 2020-07-03 21:12:14 +0800
comments: true
categories: 
---

> 作者：Yuyu He,1, Lei Zhang,1, Zhemin Yang1, Yinzhi Cao2, Keke Lian1, Shuai Li1, Wei Yang3, Zhibo Zhang1 Min Yang1, Yuan Zhang1, Haixin Duan4
> 
> 单位：1: Fudan University, 2: Johns Hopkins University, 3: University of Texas at Dalla, 4: Tsinghua University
> 
> 会议：S&P'20
> 
> 链接：[pdf](https://www.computer.org/csdl/pds/api/csdl/proceedings/download-article/1j2LfZlaQeI/pdf?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJjc2RsX2FwaSIsImF1ZCI6ImNzZGxfYXBpX2Rvd25sb2FkX3Rva2VuIiwic3ViIjoiYW5vbnltb3VzQGNvbXB1dGVyLm9yZyIsImVtYWlsIjoiYW5vbnltb3VzQGNvbXB1dGVyLm9yZyIsImV4cCI6MTU5MTY2NTg0NX0.6c-jvphgtwbuxxRIKtXc1B2RIHznD6vtb1A2ixCJi-g)

## 简介

Exerciser是指经常用于动态分析框架中的，驱动Android应用程序到达不同代码分支的应用程序。例如Monkey，可以为Android app随机生成UI事件，以达到更高的代码覆盖率。

文本输入的Exerciser有两个挑战：1）无法在合理的时间内对输入进行枚举；2）输入通常需要满足非平凡的约束。

作者提出了一种基于feedback驱动的文本输入的exerciser。使用的一个重要观察是输入无论是违反了客户端还是服务器的约束，都会提供一个线索，使用这个提供的线索进行改进输入。流程是：

1. TextExerciser根据输入返回的信息提取与格式错误的文本输入相关的所有提示。
2. 采用自然语言处理将提示解析为语法树并理解语义。
3. 根据提示语义生成约束，并使用约束求解器输出可能的输入。
4. 提交可能的输入——如果输入仍然不能满足约束条件，则TextExerciser将迭代该过程，直到找到有效输入为止。

作者从Google Play收集上下载了6,000种流行应用进行TextExerciser的评估，与现有工作相比获得了更高的代码覆盖率，并且与动态分析框架结合进行评估，能挖掘到更多的漏洞。

<!-- more -->

##  A MOTIVATING EXAMPLE

Yippi是一个下载了100,000次的消息应用，它存在一个bug：虽然它使用的都是HTTPS协议进行传输数据，但是在登录账户后，可以发送HTTP协议进行更改账户密码。

在Yippi的注册阶段有以下约束条件：

1. 用户名唯一性。与应用程序数据库中的其他用户进行比较时，“用户名”字段的输入必须唯一。不唯一会返回提示“该用户名已被使用，请尝试其他”。

2. 长度要求。 “密码”长度至少为6个字符：如果条件不满足，Yippi还将显示提示。 
3. 联合依赖。 “确认密码”的输入需要将其与“密码”字段相匹配，从而导致联合字段约束。如果这两个字段不匹配，Yippi还将警告用户。 
4. SMS身份验证。在初始注册页面之后，Yippi要求输入通过SMS发送的验证码。

现有的工具（如，Monkey），都无法exercise Yippi并触发漏洞。 Monkey将停止在注册页面，无法在登录界面之外的范围内运行该应用程序。其余工具都是依赖预先定义的输入或第三方登录，无法为Yippi生成有效的输入，因为约束很复杂且Yippi不支持任何第三方的登录。例如，预先定义的用户名在Yippi数据库中被其他人先使用，并且预先定义的密码也可能无法满足特定要求。

## METHODOLOGY

### System Workflow

![img](/images/2020-07-03/fig2.png)

1. TextExerciser提取应用程序UI中的所有文本（步骤1）
2. 通过基于学习的方法识别静态提示，并通过基于结构的差异分析识别动态提示
3. 将提示分类为不同的类别
4. 为每个提示生成语法树
5. 将生成的树转换为约束表达式
6. 通过将约束输入到求解器（例如Z3）中来生成具体输入
7.  将生成的输入反馈给目标Android应用并提取反馈，例如成功或者新的提示

### Hint Extraction 

动态提示通过差分分析获得，包括图3中两种形式，(a)页面的变化，(b)弹出的消息。

静态提示通过学习机器学习的方法提取，作者训练了一个神经网络的分类器，positive样本是提取的动态提示，negative样本是没有任何输入时的页面提示。


![img](/images/2020-07-03/fig3.png)

TextExerciser采用两种方法将提示和输入框匹配。（a）输入框和提示有相同的单词；（b）输入框和提示框的最短距离。

![img](/images/2020-07-03/fig4.png)

### Hint Parsing

**1）提示分类**

作者手工分析了Google Play上top 1,200的免费非游戏应用，将预定义了提示的分类：4个主要，10个次要和18个子次要。

![img](/images/2020-07-03/tab1.png)

1. 精确的单框。 提示准确描述了要求，例如文本输入的长度。
2. 模糊的单框。 提示含糊地描述了要求，例如，长度太小或输入内容包含无效字符。
3. 精确的联合框。 提示表示两个输入字段相关联，例如，最高工资字段的值应大于最小值字段。
4. 模糊的联合框。 表示两个字段之间的模糊关联，例如，电话号码的长度取决于另一个国家/地区的字段。

作者手工标记了1,548条提示，训练CNN和RNN的多分类器。

**2）语法树的生成**

作者对提示进行两步预处理：

(1) 删除多余的句子，如“Oops! Password must be between 7 and 15 characters. Please try again.”中删除 “Oops!” 和 “Please try again.” 

(2) 单词规范化，如用相应的数字和复数单词将拼写数字替换为相应的单数

然后使用 Stanford parser 生成抽象语法树。

**3）约束表达式**

遍历规则被定义为对带有三个谓词的语法树的查询，select指定输出节点，match是select的条件，

![img](/images/2020-07-03/travel_rule.png)

![img](/images/2020-07-03/fig5.png)

假设遍历规则属于长度约束类别，即，指定文本字段的长度需要大于阈值。 遍历需要找到一个**数字**作为阈值和一个**主体**，例如密码和日期。所以，遍历规则返回一个 Cardinal Number Node (CD) cd1 作为阈值， Noun Phrase (NP) np1 作为输入字段的主体，需要满足First()是一个NP node，cd1是一个量词短语qp1（Quantiﬁer Phrase)的孩子节点，并且qp1时一个名词（None Node）的兄弟节点。所以在图5中，(a) np1时password，cd1是6；(b) np1是username， cd1是3。

```
Select    cd1, np1
Match     Follow(qp1=QP,NN) and Contain(qp1,cd1=CD) and First(np1=NP) 
Generate  LengthConstraint(Subject(np1), Range(cd1, inﬁnity)) 
```

生成的约束有三类：

1. 长度约束。 LengthConstraint(A,Range(CD,Infty))

2. 内容约束。ContentConstraint(A,Format(NN))
3. 取值约束。这样的约束限制了输入值的范围。 它包含一个表示有效值范围。

实际使用时会简化select段，因为select被包含在Match里了。根据表I中的唯一提示编写了57条遍历规则，例子如表2所示。

![img](/images/2020-07-03/tab2.png)

### Input Generation Engine

此阶段从生成的输入，包括两个步骤：

1. 获取约束表达式中每个变量具体的值。有两个主要来源：**外部资源**和**其他字段**。外部资源包括电子邮件和文本消息，其中TextExerciser将预先注册几个电子邮件帐户和电话号码以接收诸如PIN之类的值。其他字段主要是在有联合输入要求的情况下。
2. 约束求解。使用Z3StrSolver进行约束求解。第5-6行是长度约束。第8-13行是内容约束，其中TextExerciser排除某些字符，例如提示中出现的字符。15-16行是排除旧输入。

![img](/images/2020-07-03/fig6.png)

##  EVALUATIONS

### 和现有工具的对比

作者将TextExerciser应用于Monkey， Stoat和DroidBot，并与使用它们提出时的默认设置的方法进行了对比。

![img](/images/2020-07-03/tab3.png)

1）实验配置

​	测试集包括40款App，如表4所示。

​	实验设备是四台具有相同设备配置的OnePlus 6T手机（Android 9.0，系统内部版本号A6010 41 181115）。通过ADB和Windows Server2018或者Ubuntu 16.04服务器连接。实验运行时长为1小时，重复3遍减少随机性。

​	覆盖率的检测是使用的Ella或miniTracing检测，通过触发method和activity的数量作为指标。

![img](/images/2020-07-03/tab4.png)

2) 覆盖率

结果如表5所示，平均覆盖率都有了显著的增加。

Stoat + TextExerciser触发的方法的数量有时小于Stoat + Random触发的方法的数量，尤其是在文本输入约束较少且易于实现的情况下。小幅下降的原因是，用Ruby编写的Stoat需要通过一个速度较慢，效率不高的命令行管道与用Python编写的TextExerciser通信。在一小时测试中，这种不必要的开销有时会影响代码覆盖率。特别是，每个应用程序的Stoat + TextExerciser开销平均为8.3分钟，是Monkey + TextExerciser（3.1分钟）和DroidBot + TextExerciser（3.4分钟）的两倍。

![img](/images/2020-07-03/tab5.png)

### 对已有的动态分析的改进

作者把TextExerciser应用到TaintDroid和ReCon上评估效果。

TaintDroid是一个动态污点分析框架，跟踪敏感API之间的数据流。ReCon检测报文里的隐私信息。

作者实现了基于keyword的流量分析，在网络报文中搜索如表6的关键词。

![img](/images/2020-07-03/tab6.png)

1）实验设置

因为两个工具的需要，实验采用了Android版本4.3的两个Nexus 4手机。由于DroidBot不能用于runtime的环境，这部分只使用Monkey和Stoat。 同样是执行时间1小时。

2）隐私信息泄露检测结果

从结果来看，使用TextExerciser都检测出了更多的隐私泄露。

![img](/images/2020-07-03/fig7.png)

TaintDroid检测到最多的隐私泄漏，因为与ReCon和基于关键字的搜索相比，TaintDroid可以检测到加密的标识符。相反，ReCon和基于关键字的搜索可检测更多类别的私人信息，尤其是用户credential，因为很难在污点分析中标记这些信息的来源。与基于关键字的搜索相比，Recon可以检测到更多的用户标识。ReCon的误报率较高，这主要是由于ReCon在我们的手动检查过程中经常错误地将大端口号视为zipcode。

![img](/images/2020-07-03/tab10.png)

###  在流行应用上的表现

**1）数据集和实验配置**

Google Play的6,000个随机收集的流行Android应用程序。

在16个具有4个CPU内核，2 GB RAM和2 GB SD卡的官方Android x86仿真器以及之前Android 9.0的物理设备（OnePlus 6T）上进行了该实验。首先在Android模拟器上与Monkey + TextExerciser或DroidBot + TextExerciser一起运行一个应用程序，如果执行失败，在物理设备中运行该应用程序。每次执行限制在30分钟内。

Monkey / DroidBot + TextExerciser总计分析5,640，即所有Android应用程序的94.0％，这与单独运行Monkey / DroidBot的结果相同。换句话说，TextExerciser不会将任何其他失败的应用程序带到Monkey或DroidBot。

5,060个应用程序在Android模拟器中正常运行，另外580个应用程序在物理设备中正常运行。前者失败的主要原因是aapt不会提取完整的启动器信息，因此Monkey / Droidbot无法通过Android ADB自动启动应用程序。

**2）语法规则覆盖**

TextExerciser从所有应用程序上提取了328,282条语句，输入模型后标记出其中3,450个输入提示，成功分析了其中3,012个（87.3％），生成了3,993个约束，平均每个约束5.7行代码。

分析失败的情况：(1) 提示模糊，"身份验证失败"； (2) 提示文本嵌入图中；(3) 弹出窗口消失过快，没有来得及捕捉。

提示的分布状况如图9所示。最多的三类是：C14（“非方向性约束”，例如邮件地址格式不符合要求），C1（“输入长度的下限”）和C15（“等效约束”，例如确认密码必须和密码输入相同）

![img](/images/2020-07-03/fig9.png)

**3）代码覆盖率**

在数据集上，触发activity情况如图10所示，横杠表示平均值。

 DroidBot+TextExerciser 和DroidBot+Predeﬁne虽然都是100%，但是DroidBot+TextExerciser的绝对值是72，而后者是52。

![img](/images/2020-07-03/fig10.png)

作者进一步地随机选择了10个app，分别展示了不同类别地识别情况。横杠表示中位数，条形是25%-75%。每个类别都是TextExerciser更好。

![img](/images/2020-07-03/fig8.png)

**4）生成合法输入所需次数**

第一次就能生成合法输入地次数地95.1%，第二次是96.7%。大部分只需要三轮输入的生成，所有尝试都被限制在30次以内，只有1.2%的变异超出了这个限制。作者分析了失败的原因：

（i）特定的语义提示。比如说，邀请码。

（ii）宽松约束的提示。有个应用对于手机号的输入框提示只有“请输入有效的10位数字”，但是实际上必须输入国家地区代码才能正常使用。

（iii）错误分类的提示。机器学习模型的分类错误。

**5）分类器的performance**

作者使用了开源的多分类 CNN-RNN 模型， accuracy, precision和recall 分别是 90.2%, 89.4% 和90.2%。作者认为这个误报对性能影响不大，因为也只是在增加了更多的约束而已。漏报影响比较大，但是召回率是足够的。

## Discussion

1. 支持的语言为英语，但是作者测试了10个非英语的app，其中20/21个提示可以正确处理。
2. 使用WebView的App，作者使用UiAutomator获取提示，23/25个widget可以正常获取。
3. 服务器和客户端的验证。作者在断网情况下进行测试，86/649可以进行提示。
4. 提示混淆。提示如果以图片和视频方式显示，没有办法处理。也是这个原因没有进行测试游戏应用。