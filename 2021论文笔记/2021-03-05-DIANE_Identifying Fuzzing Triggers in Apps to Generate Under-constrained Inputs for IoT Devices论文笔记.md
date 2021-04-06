# DIANE: Identifying Fuzzing Triggers in Apps to Generate Under-constrained Inputs for IoT Devices

> 作者：Nilo Redini, Andrea Continella, Dipanjan Das, Giulio De Pasquale, Noah Spahn, Aravind Machiry, Antonio Bianchi, Christopher Kruegel and Giovanni Vigna
>
> 单位：UC Santa Barbara, University of Twente, Purdue University
>
> 会议：IEEE S&P 2021
>
> 链接：https://conand.me/publications/redini-diane-2021.pdf
>

## 0 ABSTRACT

作者针对目前IoT fuzzing的弱点，提出了一种生成模糊输入的新方法，即通过companion app中的位于输入验证之后、数据转换之前的fuzzing trigger来生成有效的模糊输入。作者将该方法实现为DIANE工具，并使用了11款流行的IoT设备对其进行评估，结果表明通过识别fuzzing trigger并使用其生成模糊输入，能够有效的发现IoT设备漏洞。

<!-- more -->

## 1 INTRODUCTION

目前针对IoT设备的自动化漏洞发现工作主要存在两方面的问题：
- 由于提取和仿真固件较为困难，通常采用黑盒模糊测试的自动化分析方案，但该方法会产生大量的无效输入。
- 另一种方法是利用companion apps来生成更符合的模糊输入，但会受到应用端验证代码的约束。

作者提出了一种生成模糊输入的新方法，即通过companion app中的fuzzing trigger来生成有效的模糊输入，fuzzing trigger指的是在数据编码功能（网络序列化）之前、输入验证代码之后执行的函数，它能够生成不受应用验证代码约束的输入，并且也不会使IoT设备报错。

本文的主要贡献如下：
- 作者提出了一个识别fuzzing triggers的方法，这些fuzzing triggers在执行过程中产生合法的输入，提高IoT fuzzing效率。
- 作者实现了一个自动化的黑盒模糊测试工具DIANE。
- 作者使用了11款流行的IoT设备对工具进行评估，结果表明通过识别fuzzing trigger并使用其生成模糊输入，能够有效的发现IoT设备漏洞。作者一共发现了5台设备中的11个漏洞，其中9个漏洞是之前没有发现的。

## 2 MOTIVATION

![](/images/2021-03-05/fig1.png)

- 需要使用结构化的数据与IoT设备进行通信（prepare_msg）
- 需要通过应用端验证代码的约束，即pwd禁止包含&和'。
- 作者通过观察得到模糊输入变异必须要在编码函数之前、验证函数之后进行。太过靠近UI会导致fuzzing无效，因为之后会出现验证逻辑；太过靠近网络传输函数会导致生成的输入对设备无效，因为可能会略过一些消息编码函数。
- SendMsg可以作为一个fuzzing trigger。


## 3 APPROACH

![](/images/2021-03-05/fig2.png)

DIANE的具体实现主要分为两部分：fuzzing trigger的识别与fuzzing。DIANE仅限于IoT设备与companion apps的本地通信，尽管一些设备从云端接收命令，但研究表明，其中绝大多数（95.56%）也允许某种形式的本地通信。

### A. Fuzzing Trigger Identification

给定从输入源（例如，从UI接收的数据）到通过网络发送数据的函数的执行跟踪，**fuzzing trigger被定义为dominate所有数据转换函数和post-dominate所有输入验证函数的函数。** 作者认为追溯中的第一个数据转换函数是一个有效的模糊触发器，因为它支配着所有其他数据转换函数（包括它本身）。


> 调用图理论中dominance概念：
> 当所有从入口节点到节点n的路径都需要经过节点d，则称d dominates n。
> 当所有从n到出口节点的路径都需要经过节点p，则称p post-dominates n。


![](/images/2021-03-05/alg1.png)

fuzzing trigger识别算法由四个步骤组成。

(1) Step 1: sendMessage Candidates Identification.

识别companion app实现与设备通信逻辑的函数（sendMessage）。

- 通过观察，作者发现companion apps都包含border函数，这些函数**位于应用核心功能和外部组件（Android框架或native库）之间**。在执行时，这些函数会触发发送消息到IoT设备。因此，作者将这些border函数视为sendMessage函数。
- 作者首先通过静态分析app，收集了所有执行对native函数调用或对Android框架网络I/O函数（java.net.\*, javax.net.\*, android.net.\*）的调用的方法。

(2) Step 2: sendMessage Validation.

- 首先，作者动态的hook所有候选函数并运行app，检查当执行和阻塞相关函数时是否存在网络流量，并收集函数执行与观察到网络活动之间的时间差。
- 然后作者利用K-mean算法对时间差进行聚类，k=2，只选择平均时间差最小的集群中的sendMessage候选。

(3) Step 3: Data-Transforming Function Identification.

- 应用一般会在sendMessage函数之前执行数据转换，而fuzzing triggers是在数据转换之前的函数（否则会产生对设备无效的输入）。
- 对于数据转换函数，其得到的数据可能会被sendMessage引用，作者采用了以下方法对此类函数进行识别：
    - 首先作者静态分析了所有可能的变量，这些变量保存了sendMessage发送的数据，并且分析了这些变量可能的赋值位置。
    - 然后作者执行了一个静态过程间后向切片，从可能的变量赋值位置到从UI对象中提取值的函数，通过熵值计算来确定数据转换函数。

(4) Step 4: Top-Chain Functions Collection.

- 如果存在多个有序的数据转换函数，则构建一条数据转换链，并将链中的第一个函数称为top-chain函数。
- 如果控制了top-chain函数的变量，就可以控制发送到IoT设备的数据，并且该数据既有效也不会被输入验证影响。
- 作者通过之前建立的数据转换函数树，来识别不受任何其他函数支配的top-chain函数，并将这些函数视为fuzzing triggers。（如果没有数据转换函数，则将sendMessage视为fuzzing triggers。）


### B. Fuzzing

经过上述步骤能够得到一组fuzzing triggers，作者将其作为fuzzer的输入。

(1) Test Case Generation.

对于每个fuzzing trigger，作者通过改变其参数来生成一组测试用例，包括更改参数字符串长度、更改参数数值、采用空值、更改数组长度等策略。

(2) Identifying Crashes.

DIANE能够自动分析设备响应来识别崩溃。具体来说，DIANE首先会正常运行应用，并监视设备的响应。之后在fuzzing阶段，DIANE同样会监视应用与设备之间的流量，如果满足以下条件，则认为输入可能会导致崩溃：

- 连接中断
- HTTP Internal Server Error (500)
- 不规律的网络流量。作者认为设备在崩溃时会暂时对companion app不可用，从而会大幅度减少数据交换量。实验中，当交换的数进行小于50%，则认为设备发生故障。
- 心跳监测。在fuzzing过程中，作者不停的ping设备并监视响应，如果响应时间高于Tp (10s)，则认为发生故障。


## 4 EXPERIMENTAL EVALUATION

### A. Dataset & Environment Setup

数据集包含：11台IoT设备。

![](/images/2021-03-05/tab1.PNG)

实验环境：
- 在运行ubuntu16.04的8核128gbram机器上部署了DIANE
- 在运行Android 8.0的Google Pixel和google nexus 5x上运行companion apps
- 手机、IoT设备和运行DIANE的机器连接到同一子网
- 手动执行其初始设置、注册阶段

### B. Fuzzing Trigger Identification

![](/images/2021-03-05/tab2.PNG)

表2显示了DIANE在fuzzing trigger识别阶段的结果，作者通过人工逆向来进行验证（一位专家对每个app进行大约5小时的分析）。

> FP指的是误报，NC指的是无法确认的情况。

- sendMessage识别
    - 如果：i）在companion app调用sendMessage函数时存在网络流量，ii）函数的代码和语义指示了网络功能，则认为sendMessage函数识别正确。
    - 设备2、6、9：TP减少是因为存在函数wrapper的情况。
    - 作者检查之后没有发现漏报。
    - sendMessage误报的原因：其他的border函数包含对native方法的调用，而这些方法是在sendMessage内部或之前调用的，因此运行时间上是接近的。

- fuzzing trigger识别
    - 设备4、5、9：fuzzing trigger为sendMessage函数，即没有数据转换函数，或数据转换函数包含在网络收发函数中。

- 总体上，存在5个误报，原因包括：i) 找不到数据转换函数的调用函数，因此将其错误的识别为top-chain函数；ii) 阈值问题。
    - 误报并不影响DIANE的有效性，而是会影响其效率（花费的时间）。


### C. Vulnerability Finding

![](/images/2021-03-05/tab3.png)

如表3所示，作者在5台设备上发现了11个问题，并向相应的厂商报告，所有问题已被修复。

作者还将DIANE与IoTFuzzer进行了比较：

- IoTFuzzer需要人工预处理以检测不同的设备和companion apps，包括i) 将hook的函数限制为Java包的子集；ii) 人工指定加密函数。
- DIANE在除了设备3、6（结果一样）之外的其他设备上的性能都优于IoTFuzzer，可能的原因包括IoTFuzzer无法正确识别发送函数，或无法产生能够导致崩溃的输入（由于应用端验证代码）。

此外，作者将DIANE与一些network fuzzer进行了比较，结果表明，network fuzzer无法发现任何bug，并且大部分的network fuzzer都无法处理相应IoT设备的网络协议。

### D. App-side Sanitization

作者对应用中的输入验证代码进行了评估，发现在数据集中的11个apps中至少7个包含输入检查。

- 设备3、9、10：无法跟踪从sendMessage到UI对象的数据流，作者认为这是由于逆向工具不精确导致的。

- 作者进一步地对应用端验证代码进行了大规模的研究（2081个IoT companion apps），约51%的应用会对发送的数据进行验证。

## 5 LIMITATIONS

- 如果应用端验证检查在数据转换函数或sendMessage函数中实现时，DIANE无法绕过这些检查。
- DIANE代码覆盖率有限，无法识别应用程序中未执行的fuzzing trigger。
- DIANE目前无法模糊嵌套的Java对象。

## 6 CONCLUSIONS

本文针对IoT Fuzzing主要解决了两方面的问题：
- companion apps中的应用端验证逻辑可能会降低fuzzing效率
- 如果不考虑数据编码函数，可能会产生对设备无效的输入

作者提出了一种介于网络级模糊测试和UI级模糊测试之间的新方法，核心思路是识别位于输入验证之后、数据转换函数之前的fuzzing trigger，并以此生成模糊输入。作者在DIANE实现了该方法，并在11个不同品牌的真实物联网设备上进行了评估。