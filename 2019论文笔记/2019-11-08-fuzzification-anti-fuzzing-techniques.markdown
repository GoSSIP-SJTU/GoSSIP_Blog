---
layout: post
title: "FUZZIFICATION: Anti-Fuzzing Techniques"
date: 2019-11-08 00:30:55 -0500
comments: true
categories: 
---
> 作者：Jinho Jung, Hong Hu, David Solodukhin, and Daniel Pagan; Kyu Hyung Lee; Taesoo Kim
>
> 单位：Georgia Institute of Technology, University of Georgia
>
> 会议：USENIX Security Symposia 19
>
> Paper：https://www.usenix.org/system/files/sec19-jung.pdf
<hr/>

## 1 Introduction

模糊测试作为一种软件测试技术，也可以被恶意攻击者使用以发现零日漏洞。因此，**开发人员希望在其产品上应用反模糊技术以阻止攻击者进行模糊测试，其概念类似于使用混淆技术削弱逆向工程。**

在本文中，作者提出了一种**二进制保护的新方向，称为FUZZIFICATION**。攻击者仍然可以从受FUZZIFICATION保护的二进制文件中查找错误，但需要花费更多的精力（例如，CPU，内存和时间）。因此，获得原始二进制文件的开发人员或其他受信任方能够在攻击者之前检测程序错误并合成补丁。

<!--more-->

有效的模糊化技术应启用以下三个功能：

* 首先，它应该有效地阻止现有的模糊测试工具，使其在固定时间内发现更少的错误；
* 其次，受保护的程序在正常使用下仍应有效运行；
* 最后，不应被分析技术轻易地识别保护代码或将其从受保护的二进制文件中删除。

现有技术无法同时实现这三个目标：

* **software obfuscation techniques**：通过混淆二进制表示来阻止静态程序分析的软件混淆技术似乎在阻止fuzzing尝试方面很有效。但它有两方面的不足：
  1. 混淆给正常程序执行带来较大的开销，使用Obfuscator-LLVM时则会使执行速度降低25倍。
  2. 混淆处理在路径探索方面不能有效地阻止模糊测试。它可以减慢每个模糊执行的速度，但是每次执行的路径发现几乎与模糊原始二进制文件的路径发现相同。
* **software diversification**：软件多样化能缓解攻击但无法从攻击者分析中隐藏原始漏洞。

在本文中，作者为开发人员提出了**三种FUZZIFICATION技术**，以保护程序免受恶意模糊测试侵害：**SpeedBump（注入delay），BranchTrap（插入jumps）和AntiHybrid（阻止其他分析技术在 fuzzing 的应用）**。作者**开发了相应的防御机制**，以阻止攻击者从受保护的二进制文件中识别或删除该技术。

为了评估FUZZIFICATION技术，作者将它们**应用于LAVA-M数据集和九个实际应用程序**，包括libjpeg，libpng，libtiff，pcre2，readelf，objdump，nm，objcopy和MuPDF。然后，作者**使用四个流行的fuzzer**（AFL，HonggFuzz，VUzzer和QSym）在相同的时间内测试原始程序和受保护程序。

在本文中，作者做出了以下贡献：

* 阐明了**anti-fuzzing方案的新研究方向**，即所谓的FUZZIFICATION
* 开发了**三种FUZZIFICATION技术**来减缓fuzzing速度、隐藏路径覆盖范围以及阻止动态污点分析和符号执行。
* **根据流行的fuzzer和通用基准评估以上技术**。实现的技术阻碍了这些fuzzer，从真实二进制文件中发现的bug减少了93％，从LAVA-M数据集中发现的bug减少了67.5％，在保持用户指定的开销预算的同时，覆盖率也降低了70.3％。且数据流和控制流分析技术无法轻易移除FUZZIFICATION技术。
  源代码：https://github.com/sslab-gatech/fuzzification

## 2 Background and Problem

### 2.1 Fuzzing Techniques

这里引用‘A systematic review of fuzzing techniques’文中的fuzz的框架图对现代fuzz结构进行简要说明。

<center>
<img src="/images/2019-11-08/fuzz_structure.png">
fig-0 An architectural diagram of fuzzing system</center>

Fuzzing的效率提升一般有两种方式：1. 每次执行速度更快；2. 需要更少的执行次数。

#### 2.1.1 Fuzzing with Fast Execution

提高模糊测试效率的一种直接方法是使每次执行速度更快。当前的研究重点在于几种快速执行技术，包括：1. 定制的系统和硬件以加速fuzz计算执行；2. 并行fuzzing以大规模摊销绝对执行时间。 

#### 2.1.2 Fuzzing with Coverage-guidance

Coverage-guided fuzzer为每次fuzzing执行收集代码覆盖率，并优先对触发新覆盖率的输入进行fuzz。大多数流行的fuzzer都以code coverage为指导，例如AFL，HonggFuzz和LibFuzzer，但是覆盖率表示和覆盖率收集的方法不同。

**Coverage representation**。大多数fuzzer都采用基本块（basic blocks）或分支来（branches）表示代码覆盖率。例如，HonggFuzz和VUzzer使用基本块覆盖率，而AFL则考虑分支覆盖率。 Angora 将分支覆盖范围与调用堆栈结合在一起，以进一步提高覆盖范围的准确性。但更细粒度的覆盖范围会给每次执行带来更高的开销，并且会损害模糊测试的效率。

**Coverage collection**。Fuzzer通常维护自己的数据结构来存储覆盖率信息。例如，AFL和HonggFuzz使用固定大小的数组，而VUzzer使用Python中的Set数据结构存储其覆盖范围。但是，结构的大小是精度和性能之间的折衷：太小的内存无法捕获每个覆盖范围的变化，而太大的内存会带来很大的开销。例如，如果位图大小从64KB更改为1MB，则AFL的性能会降低30％。

#### 2.1.3 Fuzzing with Hybrid Approaches

首先，fuzzer不区分具有不同类型（例如，magic number, length specifier）的输入字节，因此可能浪费时间来对不影响任何控制流的次要字节进行突变。 污点分析帮助查找哪些输入字节用于确定分支条件。通过关注这些字节的突变，fuzzer可以快速找到新的执行路径。 其次，模糊测试器无法轻松解决复杂的条件(e.g., 与Magic value或校验和进行比较)。 利用符号执行可以解决这个问题，但产生了高昂的开销。

### 2.2 FUZZIFICATION Problem

需求场景：程序开发人员希望由自己或由受信方公开漏洞，而非恶意用户。Anti-Fuzzing技术可以通过遏制来自恶意攻击者的fuzzing来帮助实现这一目标。

FUZZIFICATION的工作流程如Fig2所示。开发人员将其代码编译为两个版本：(1)使用FUZZIFICATION技术进行编译以生成受保护的二进制文件;(2)通过常规方式进行编译以生成普通的二进制文件。
在FUZZIFICATION技术的保护下，恶意攻击者无法迅速找到许多bug。而受信方可以以本机速度启动正常二进制文件的模糊测试，从而可以及时发现更多错误并修复。

<center>
<img src="/images/2019-11-08/fuzzification_2.png">
Figure 2: Workflow of FUZZIFICATION protection
</center>

#### 2.2.1 Threat Model

攻击者模型设定：

* 尝试通过最新的fuzzing技术来发现软件漏洞
* 具有有限的资源，如计算能力（至多与受信任方相似的资源）
* 具有受FUZZIFICATION保护的二进制文件，并且了解FUZZIFICATION技术。
* 可以访问不受保护的二进制文件甚至是程序源代码（例如，内部攻击者或通过代码泄漏）的攻击者不在研究范围内。

#### 2.2.2 Design Goals and Choices

FUZZIFICATION技术应同时实现以下四个目标：

* **Effective**：与原始二进制文件相比，它应有效减少在受保护二进制文件中发现的bug数量。
* **Generic**：它解决了fuzzing的基本原理，通常适用于大多数fuzzer。
* **Efficient**：它为常规程序执行带来了的额外开销较小。
* **Robust**：它可以抵抗试图从受保护的二进制文件中删除它的对抗分析。

考虑到这些目标，作者检查了四种阻止恶意模糊测试的设计选择，都不能满足所有目标。

<center>
<img src="/images/2019-11-08/fuzzification_3.png">
</center>

### 2.3 Design Overview

作者提出了三种FUZZIFICATION技术（SpeedBump，BranchTrap和AntiHybrid）来针对第2.1节中讨论的每种技术。

1. **SpeedBump**将细粒度的延迟原语（delay primitives）注入cold path中（fuzz会频繁执行这些path的但普通执行很少使用）。
2. **BranchTrap**构造了许多对输入敏感的分支，以诱使coverage-based fuzzer将其精力浪费在毫无结果的路径上。同样，它有意地使代码覆盖区存储充满频繁的路径冲突，从而使fuzzer无法识别触发新路径的有趣输入。
3. **AntiHybrid**将显式数据流转换为隐式数据流，以防止通过污点分析进行数据流跟踪，并插入大量伪造符号以在符号执行过程中触发路径爆炸。

<center>
<img src="/images/2019-11-08/fuzzification_4.png">
<div align="center">
Figure 3: Overview of Fuzzification process
</div>
</center>

图3展示了FUZZIFICATION系统。输入：**程序源代码、一组常用测试用例以及开销预算**，输出：**FUZZIFICATION技术保护的二进制文件**。大致步骤如下：

1. 编译生成正常的二进制文件，运行测试用例收集基本块频率，找到正常执行很少使用的基本块。 
2. 根据配置文件将三种FUZZIFICATION技术应用于该程序并生成临时的受保护二进制文件。 
3. 用正常测试用例再次测临时二进制文件的开销。如果开销超出预算，则返回到步骤2，以减少程序的slowdown。如果开销远低于预算，我们会相应增加开销。
4. 开销极限逼近预算，将生成受保护的二进制文件。

## 3 SpeedBump: Amplifying Delay in Fuzzing

<!-- 作者提出了一种称为SpeedBump的技术，以减缓fuzzed execution的速度，同时将对正常执行的影响降至最低。 -->

* **依据**：fuzz执行通常会进入诸如错误处理之类的路径，而这些路径通常不会被正常的执行访问。在cold path中注入延迟会大大降低fuzz执行的速度，但不会对常规执行产生太大影响。
* **方法**：首先从测试用例中确定cold path，然后将延迟注入到最少执行的代码路径。工具自动确定注入延迟的路径数量以及每个延迟大小，使正常执行过程中受保护的二进制文件产生的开销在用户定义的预算内。

  * **Basic block frequency profiling.** FUZZIFICATION生成一个基本的块频率曲线，以识别冷路径。
  * **Configurable delay injection.** 重复操作，确定要注入延迟的代码块及每个延迟大小，逼近开销。

### 3.1 Analysis-resistant Delay Primitives

由于攻击者可使用程序分析来识别和删除简单的延迟原语（例如，调用sleep），因此作者设计了基于CSmith的鲁棒性的原语。这些原语涉及算术运算并与原始代码库相关联。

<center>
<img src="/images/2019-11-08/fuzzification_5.png">
<div align="center">
</div>
</center>

**Safety of delay primitives.** 作者利用CSmith和FUZZIFICATION的安全检查来确保生成的代码没有错误。
**Fuzzers aware of error-handling blocks.** VUzzer和T-Fuzz等fuzzer，通过概要分析来识别错误处理基本块，并将它们从代码覆盖率计算中排除，以避免重复执行。该技术使用类似的配置步骤来识别冷路径，可能会影响SpeedBump技术的有效性。但SpeedBump的冷路径不仅包括错误处理基本块，而且还包括很少执行的功能块。**FUZZIFICATION将集中于检测很少执行的功能块**，以最大程度地发挥其作用。

## 4 BranchTrap: Blocking Coverage Feedback

Fuzzer使用code coverage来找到有趣的输入并优先排序。如果插入大量条件分支，且分支条件对轻微的输入变化敏感，那么当fuzzing陷入这些分支陷阱时，基于覆盖的fuzzer将浪费资源来探索大量毫无价值的路径。因此，作者提出了BranchTrap的技术，通过误导或阻止覆盖反馈来欺骗基于覆盖的模糊器。

### 4.1 Fabricating Fake Paths on User Input

BranchTrap的**第一种方法是构造大量条件分支和间接跳转**，并将其注入到原始程序中。每个伪造的条件分支都依赖于某些输入字节来确定是否采用该分支，而间接跳转则根据用户输入来计算其目标。为了有效地使模糊测试者关注fake branches，作者考虑以下四个设计方面：

* BranchTrap应该制造足够数量的伪造路径以影响模糊测试策略。
* 注入的新路径为常规执行带来了尽可能小的开销。
* BranchTrap中的路径应该是根据用户输入确定的，因为某些fuzzer可以检测并忽略不确定的路径。
* BranchTrap无法轻易被对手识别或删除。

#### 4.1.1 BranchTrap with CFG Distortion

BranchTrap的一个简单实现是注入一个跳转表，并使用一些输入字节作为访问该表的索引。为了强化BranchTrap，作者根据用户输入使每个注入的分支的返回地址多样化。受ROP的启发：通过链接各种小的代码段，将现有代码重用于恶意攻击。作者的方法可能会严重扭曲CFG，使取消BranchTrap更具挑战性。实施过程分为三个步骤：

* 首先，BranchTrap从程序汇编代码中收集函数结尾（function epilogues）。
* 其次，将具有相同指令序列的函数结尾分组到一个跳转表中
* 第三，重写程序集，以便该函数使用一些输入字节作为跳转表索引，从相应的跳转表中检索几个等效的结尾，以实现原始函数的返回。

<center>
<img src="/images/2019-11-08/fuzzification_5_1.png">
<div align="center">
Figure 6: BranchTrap by reusing the existing ROP gadgets in the original binary.
</div>
</center>

基于ROP的BranchTrap具有三个优点：

* Effective：控制流与用户输入突变一起不断敏感地变化从而引入足够数量的无效路径，使coverage feedback效果降低。且BranchTrap保证路径确定性，从而使fuzzer不会忽略这些伪造路径。
* Low overhead：轻量级的操作（XOR；解析跳转地址；跳转到gadget）为普通用户造成的开销低。
* Robust：基于ROP的设计大大增加了对手识别或修补二进制文件的复杂性。

### 4.2 Saturating Fuzzing State

BranchTrap的**第二种方法是使fuzzing状态饱和**，这会阻止fuzzer了解代码覆盖率的进度。与第一种方法引起fuzzer专注于毫无结果的输入不同，这里的目标是防止fuzzer找到真正有趣的输入。

BranchTrap能够为一些很少有人访问的基本块引入大量（例如1万到10万）确定​​性分支。一旦fuzzer到达这些基本块，其覆盖范围表将迅速填满。在之后执行中大多数新发现的路径将被视为已访问，因此fuzzer将忽略实际有趣的输入。

## 5 AntiHybrid: Thwarting Hybrid Fuzzers

混合模糊测试方法利用符号执行或动态污点分析来提高模糊测试效率。然而，混合方法具有众所周知的弱点：

* 符号执行和污点分析都消耗大量资源，从而限制了它们分析简单程序的能力。
* 符号执行受到路径爆炸问题的限制。如果处理符号需要复杂的操作，则符号执行引擎必须详尽地探索和评估所有执行状态。而且大多数符号执行引擎都无法运行到执行路径的末尾。
* DTA分析难以跟踪隐式数据依赖性。例如，为覆盖通过控制通道的数据依赖性，DTA必须在有条件分支之后将taint属性主动传播到任何变量，从而使分析更昂贵而结果不太准确。

#### Introducing implicit data-flow dependencies.

作者**将原始程序中的显式数据流转换为隐式数据流**，以防止污点分析。FUZZIFICATION首先确定分支条件和有趣的信息sink（例如strcmp），然后根据变量类型注入数据流转换代码。

<center>
<img src="/images/2019-11-08/fuzzification_5_3.png">
<div align="center">
Figure 8: Example of AntiHybrid techniques.
</div>
</center>

隐式数据流阻碍了跟踪直接数据传播的数据流分析。但它无法通过差异分析来防止数据依赖性推断。例如，最近的工作中RedQueen通过模式匹配推断输入条件和分支条件之间的潜在关系，从而可以绕过隐式数据流转换。但是，RedQueen要求在输入中明确显示分支条件值，可以通过简单的数据修改轻易地弄虚作假。

#### Exploding path constraints.

为了阻止使用符号执行的混合fuzzer，**FUZZIFICATION注入多个代码块以有意触发路径爆炸**。具体来说，通过比较原始操作数的哈希值来替换每个比较指令。采用哈希函数是因为符号执行无法轻松确定具有给定哈希值的原始操作数。由于哈希函数通常会在程序执行中引入不可忽略的开销，因此作者利用轻量级的循环冗余校验（CRC）循环迭代来转换分支条件以减少性能开销。

## 6 Evaluation

作者评估FUZZIFICATION技术，了解其在阻止fuzzer探索程序代码路径（§6.1）和检测错误（§6.2）方面的有效性，保护实际大型程序的实用性（§6.3）以及针对对抗性分析技术的鲁棒性 （第6.4节）。

FUZZIFICATION 框架共以 6,559 行 Python 代码和 758 行 C++ 代码实现。 SpeedBump 技术实现为 LLVM 传递，在编译期间将延迟注入cold blocks中。 对于 BranchTrap，分析汇编代码并直接对其进行修改。 对于 AntiHybrid 技术，使用 LLVM 通过来引入路径爆炸，并用 python 脚本自动注入隐式数据流。

作者针对可在二进制文件上运行的四个最先进的fuzzer，对FUZZIFICATION进行了评估。选择LAVA-M数据集和九个实际应用作为模糊测试目标。这9个实际程序包括Google模糊测试套件的4个应用程序，binutils的4个程序和PDF阅读器MuPDF。作者对这些二进制文件执行了两组实验，总结在表3中。

<center>
<img src="/images/2019-11-08/fuzzification_6.png" >
<div align="center">
</div>
</center>


<center>
<img src="/images/2019-11-08/fuzzification_13.png">
<div align="center">
</div>
</center>

**Evaluation metric** 使用两个指标来衡量有效性:实际路径获得的code coverage以及unique crashes。

### 6.1 Reducing Code Coverage

#### 6.1.1 Impact on Normal Fuzzers

作者评估了FUZZIFICATION对减少针对AFL-QEMU和HonggFuzz-Intel-PT的实际路径数量的影响。图9显示了AFL-QEMU在具有五个保护设置的不同程序上72小时的模糊结果。

<center>
<img src="/images/2019-11-08/fuzzification_7.png" >
<div align="center">
</div>
</center>

表5显示了每种技术对阻碍路径发现的影响。其中，SpeedBump可以最好地防御。

<center>
<img src="/images/2019-11-08/fuzzification_15.png">
<div align="center">
</div>
</center>

#### 6.1.2 Impact on Hybrid Fuzzers

<center>
<img src="/images/2019-11-08/fuzzification_16.png">
<div align="center">
</div>
</center>

作者还针对QSym评估了FUZZIFICATION对代码覆盖的影响，QSym是一种混合的模糊器，利用符号执行来帮助进行模糊处理。图10显示了QSym从原始二进制文件和受保护的二进制文件中发现的实际路径数。总体而言，使用这三种技术，FUZZIFICATION可以将路径覆盖范围平均减少QSym的80％。
**与正常的模糊结果比较**。 QSym使用符号执行来帮助在模糊测试中找到新路径，比AFL发现的真实路径多44％。通过FUZZIFICATION，QSym与普通的模糊测试器相比，优势从44％降低到12％。

### 6.2 Hindering Bug Finding

测量fuzzer从原始二进制文件和受保护的二进制文件中发现的唯一崩溃（unique crash）的数量。

<center>
<img src="/images/2019-11-08/fuzzification_9.png" >
<div align="center">
</div>
</center>

**Impact on Real-World Applications** 图11显示了72小时内三个fuzzer发现的唯一崩溃总数。FUZZIFICATION将发现的崩溃数量减少了93％，其中AFL减少88％，HonggFuzz减少98％，QSym减少94％。
**Impact on LAVA-M Dataset** 与其他经过测试的二进制文件相比，LAVA-M程序体积更小，操作更简单。所以使用了微小的延迟原语（即10μs至100μs），将基本块指令的比例从1％调整为0.1％，减少了应用的AntiHybrid组件的数量，并注入了较小的确定性分支以减少代码大小开销。FUZZIFICATION可以减少VUzzer发现的错误的56％和QSym减少的发现的78％。

### 6.3 Anti-fuzzing on Realistic Applications

<center>
<img src="/images/2019-11-08/fuzzification_11.png">
<div align="center">
</div>
</center>

为了了解FUZZIFICATION在大型现实应用中的实用性，作者选择了六个具有GUI并依赖于数十个库的程序，测量FUZZIFICATION技术的开销和受保护程序的功能。对于每个受保护的应用程序：

* 首先要使用多个输入（包括给定的测试用例）手动运行它，并确认FUZZIFICATION不会影响程序的原始功能。
* 其次，针对给定的测试用例，测量受保护二进制文件的代码大小和运行时开销。如表7所示，FUZZIFICATION平均会引入5.4％的代码大小开销和0.73％的运行时开销。


#### Anti-fuzzing on MuPDF.

作者还评估了FUZZIFICATION在保护MuPDF抵御三种Fuzzer（AFL，HonggFuzz和QSym）方面的有效性。使用和表4所示的相同参数编译了二进制文件，使用CLI界面执行了基本块分析。经过72小时的fuzzing后，**没有任何fuzzer从MuPDF中发现任何错误**。因此，作者比较原始二进制文件和受保护二进制文件之间的**实际路径数**。如图13所示，FUZZIFICATION平均将总路径减少了55％，特别是AFL减少了77％，HonggFuzz减少了36％，QSym减少了52％。

<center>
<img src="/images/2019-11-08/fuzzification_12.png">
<div align="center">
Figure 13: Path discovered by different fuzzers from the original MuPDF and the one protected by three FUZZIFICATION techniques.
</div>
</center>

### 6.4 Evaluating Best-effort Countermeasures

作者针对对手可能用来逆转保护措施的现成程序分析技术，评估了FUZZIFICATION技术的鲁棒性。

<center>
<img src="/images/2019-11-08/fuzzification_12_1.png">
</center>
## 7 Discussion and Future Work

在本节中，作者讨论FUZZIFICATION的局限性，并提出针对它们的临时对策。

* **Complementing attack mitigation system.** 
* **Best-effort protection against adversarial analysis.** 
* **Trade-off performance for security.** 
* **Delay primitive on different H/W environments.**
