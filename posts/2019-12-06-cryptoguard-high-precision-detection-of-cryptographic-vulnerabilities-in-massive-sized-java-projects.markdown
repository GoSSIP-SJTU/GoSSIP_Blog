---
layout: post
title: "CryptoGuard: High Precision Detection of Cryptographic Vulnerabilities in Massive-sized Java Projects"
date: 2019-12-06 05:26:33 -0500
comments: true
categories: 
---
> 作者：Sazzadur Rahaman¹, Ya Xiao¹, Sharmin Afrose¹, Fahad Shaon², Ke Tian¹, Miles Frantz¹, Murat Kantarcioglu², Danfeng(Daphne) Yao¹
>
> 单位：¹Computer Science, Virginia Tech, Blacksburg, VA, ²Computer Science, University of Texas at Dallas, Dallas, TX
>
> 出处：CCS'19(The 26th ACM Conference on Computer and Communications Security)
>
> 原文：[https://dl.acm.org/citation.cfm?id=3345659](https://dl.acm.org/citation.cfm?id=3345659)

---

## Abstract

密码学误用(比如硬编码的密钥，口令，有漏洞的证书验证)，会威胁软件安全。在不影响密码学误用分析质量的前提下降低分析的假阳性，这一目标很难实现，所以自动化检测大型项目(百万行代码以上)密码学误用这一目标也未实现。

作者设计了CryptoGuard，对大型java项目进行密码学误用检测，并且通过一系列优化算法，在实验中将误报率降低了76%-80%。

作者使用cryptoguard对46个大型Apache项目和6181个Android app进行了密码学误用分析。对于Apache项目，检测出1295个密码学误用，手动分析后确认了其中的1277个(98.61%)。

<!--more-->

## Introduction

现有的密码学误用分析分静态(CrySL, FixDroid, CogniCrypt, CryptoLint)和动态(SMV-Hunter, AndroSSL)两个方向。
静态分析特点是不需要程序执行，可适用于大量程序，覆盖大量安全规则，漏报率较低。动态分析特点：需要程序运行，触发并检测特定的误用情况，相对静态误报率低。大规模检测密码学误用，一般选择静态分析的方法。

现有的密码学误用静态分析工具没有针对大型项目的优化，在大型项目上使用时会crash；没有针对SSL/TLS API进行检测；没有被设计于检测复杂的误用场景。
针对这些问题，作者设计了一个支持大型项目的，支持多种规则的，高精度的密码学误用静态分析工具。

作者制定了一系列误用规则，根据具体规则选取前向或者后向的程序切片进行数据流分析，从而检测密码学误用，并通过一系列算法来排除造成误报的情况。

作者贡献：

- 设计和实现了一组用于检测密码学和SSL/TLS API误用的分析算法。实现了静态密码学误用检测工具CryptoGuard。
- 制定了16条误用规则，针对46个apache项目进行了测试，规则中的15条被检测出来，39个项目中至少存在一种密码学误用，33个至少存在两种，没有发现predictable seed的误用。
- 针对6181个android app进行了测试，结果表明95%的密码学误用来自与应用一起打包的三方库，包括来自google，facebook，apache，tencent的库。
- 编写了针对密码学误用的测试用例CRYPTOAPI-BENCH，包含112个单元测试用例，包括函数内和跨函数的误用，针对成员变量的误用，假阳性用例和正确API使用的用例。

## Threat Model

作者首先将密码学误用分为以下五类，然后针对不同类别制定了误用规则。

1. Vulnerabilities due to predictable secrets.
   - 可预测的敏感数据都是不安全的，包括硬编码的常量，或者以常量为参数的函数返回值。
2. Vulnerabilities from MitM attacks on SSL/TLS.
   - 错误使用jsse可能会导致中间人攻击。
3. Vulnerabilities from predictable PRNGs.
   - 不安全的伪随机数算法以及可预测的随机数算法种子
   - java.util.Random 不安全，LCG
4. Vulnerabilities from CPA.
   - 选择明文攻击，明文对应唯一密文，ecb模式，固定iv，PBE固定盐值
5. Vulnerabilities from feasible bruteforce attacks.
   - 可行的穷举攻击，枚举空间不足以抵抗当前算力。md5，sha1碰撞，64bit对称加密（des，3des，idea，blowfish），rc2，rc4，1024bit rsa，少于1000轮迭代的PBE密钥导出算法

作者按攻击者的能力和攻击难度将误用的严重性分为高中低三档。

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled.png](/images/2019-12-06/Untitled.png)

## Technical Challenge and Solution

密码学误用的检测，面临的挑战主要是准确度和效率两方面。

多种误用检测是针对密码学API里的常量进行检测(key, password, seed, iv)。

程序切片可以划分影响程序变量或者被程序变量影响的语句，可以追溯敏感数据是否被常量影响，但是实际使用中会有一些问题。

- 检测精度
  - 良性的常量会影响到敏感数据，被误报为密码学误用。例如状态标识符，资源标识符，数组索引值等等。
  - 一些检测工具会假设系统库和运行时库在误用分析时都会被提供，但实际上很多JAVA项目并不满足这个假设，程序切片探索不到的路径可能会导致误报。
- 效率和覆盖率
  - 针对大型项目，构建整个项目的CFG会造成很大的空间和时间上的开销。

### false positives

进行误用分析时，如果方法的主体不可见(例如一个库没有打包进项目)，程序切片无法跟进函数进行分析，从而造成假阳性。例如，Figure 1(a)里如果Context库不可见，分析时无法跟进跟进getProperty函数，所以一般的检查工具会把"pass.key"当作一个硬编码的密钥。

数据结构的特性也会造成假阳性，例如1(a)的第11行，1可能会被认为是一个硬编码密钥，因为它对keybytes产生了影响。
解决方法：优化切片输出，通过检查上下文验证"pass.key"是key的标识符；丢弃和数据结构相关的常量

### precision vs runtime tradeoff

针对百万行以上的代码，生成整个项目的CFG耗时且不是必要的。密码学函数只占项目的一小部分，但是很多现有工具都会生成整个项目的CFG。
作者采用以下方法：

1. 控制正交方法的检测深度
   常量的一个显著特征是在使用前不需要或者只需很少的处理。处理一般也是通过正交方法处理的，(即处理对常量无影响，例如编码转换)。限制探索的深度会影响检测的精度和时间，根据后续实验，作者设置深度为1。
2. 按需分析
   作者通过按需的过程间后向数据流分析来实现后向程序切片，从切片准则开始，向上按需切片(只考虑影响切片准则的语句，不考虑其他)。

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%201.png](/images/2019-12-06/Untitled%201.png)

## Crypto-Specific Slicing

作者使用静态的def-use分析以及前向和后向程序切片来检测java密码学API误用。实现时将检测策略分成多步，每一步都可以通过一轮程序切片实现。分析切片的结果可以检测误用的存在。不同的误用场景需要不同的切片方法，在Table 1中总结。通用的切片方法不够精确，之后会讨论优化的方法。

### program slice

def-use分析可以识别变量的definition和use语句，以及它们之间的依赖关系。
变量v的definition是修改v的语句，例如声明和赋值。
变量v的use是读取v的语句（例如v作为一个函数的参数）。
给定一个切片准则(一条语句或者一条语句里的一个变量)，后向程序切片用于计算数据流中影响切片准则的语句集合； 前向切片用来计算被切片准则影响的语句集合。给定一个程序和一个切片准则，一个切片器返回程序切片的列表。

### backward slicing

对于过程间后向切片，将切片标准定义为目标方法调用的一个参数，例如SecretKeySpec的key参数。对于过程内后向切片，定义三种切片准则，方法的参数，赋值操作以及return和throw，比如规则4中检查hostname verifier接受所有host，用return语句做切片。

**Intra-procedural backward slicing**
过程内后向切片，针对语句的def-use特性来确定语句是否属于某一个切片。作者使用了soot进行静态数据流分析。分析过程中，遇到正交方法调用时，会递归的对它们切片，以收集影响成员变量或者方法返回值的参数和语句。为了降低开销，会限制正交方法的探索深度.

**On-demand Inter-procedural backward slicing**

过程间分析以过程内分析为基础。1）首先建立项目内所有函数的caller-callee关系图。2）确定切片准则所在函数的调用者。3）对于所有的调用者，根据函数的调用链，递归的使用过程内分析来得到所有的过程间后向切片。4）流程是对成员变量敏感的，典型的成员变量初始化是赋值语句，分析时遇到对成员变量的赋值操作时，将递归向上分析影响成员变量的语句。

### forward slicing

**Intra-procedural forward slicing**

对于规则6和规则15使用过程内前向切片。将赋值语句当作切片准则，跟踪执行流。由于6和15的误用是在单个函数之内，所以不需要跨函数分析。

**Inter-procedural forward slicing**

过程间的前向切片一般是作为后向切片的补充。

对于figure 2，$r1的成员变量只能通过正交调用间接访问，给定一个常量，cryptoguard会确定此常量会对这个类的哪些成员变量产生影响。之后，当检测到对相同对象的赋值时，由于之前记录了成员变量会影响返回语句，cryptoguard会报告此常量。
通过这个分析，cryptoguard会知道“mytext”不是一个硬编码的key。

例如Figure 2，寻找影响key的语句时用的是后向切片，可以判断出key来自r1的一个成员变量，但是后向切片无法确认成员变量是从哪里赋值的。再利用前向切片，可以知道“mykey”被赋值到成员变量key，从而检测到误用。

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%202.png](/images/2019-12-06/Untitled%202.png)

## Refinement for FP Reduction

作者设计了一系列优化算法，以排除安全无关指令的影响，从而降低误报。作者的规则中，有八条规则是对常量或者可预测值的检测，确保机密数据不从硬编码得到或导出。但是，很多常量并不会造成安全影响，之前的检测中会将这些常量误报。根据经验，作者把这些常量分为五类，从切片中排除这些常量的影响。

**RI-I: Removal of State Indicators**

描述变量状态的常量，例如正交调用时编码转换用到的“UTF-8”。如果限制了分析深度，无法对getBytes方法的内部进行分析，def-use分析会把“UTF-8”当作影响r4的常量，报告一个误用。
`$r4 = virtualinvoke $r2.<java.lang.String:getBytes(java.lang.String)>("UTF-8")byte[]`

排除所有的虚函数调用的参数会造成误报，例如`virtualinvoke $r5.<KeyHolder: void setKey(java.lang.String)>("abcd")`.
观察到赋值语句中调用虚函数时，函数参数一般用来描述变量状态，所以作者提出：1）赋值语句中虚函数的常量参数可以被排除。2）影响这些参数的常量也可以被排除

**RI-II: Removal of Source Identifiers**

限制分析深度之后，有些资源标识符也会被误认为是有影响的。例如：
`$r30 = interfaceinvoke r29.<java.util.Map:get(java.lang.Object)>("ENCRYPT_KEY")`.
"ENCRYPT_KEY"会被认为是影响$r30的常量，但实际上这是JAVA的map数据结构检索时的标识符。
`$r4 = staticinvoke <Context:java.lang.String.getProperty(java.lang.String)>(src)`
src是外部资源检索的标识符，也会被认为影响$r4.
要排除此类误报，作者提出：排除赋值语句中的静态调用的常量参数

**RI-III: Removal of bookkeeping indices**

`byte[] iv = new byte[] {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0}`

以上代码会被转化为以下Jimple表示，硬编码的长度和索引值会被误报。作者提出排除所有影响array，list，set索引和长度的常量。

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%203.png](/images/2019-12-06/Untitled%203.png)

**RI-IV: Removal of contextually incompatible constants**

可以根据常量的上下文来排除一些误报。比如整数类型或者布尔类型的常量不会被用作key和iv，可以根据上下文排除到这一类误报

**RI-V: Removal of constants in infeasible paths**

排除掉null和空的字符串

### Evaluation of Refinement Method

作者比较了使用上述优化算法前后的不同规则的警报数量，并验证了减少的部分均为误报

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%204.png](/images/2019-12-06/Untitled%204.png)

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%205.png](/images/2019-12-06/Untitled%205.png)

## Security Findings and Evaluation

作者选择了46个流行的使用了密码学API的Apache项目，流行度是根据github上的star和fork判定的。46个项目的代码量的最大最小和平均值分别为2, 571K (Hadoop), 1.1K (Commons Crypto) ，402K，作者从这些项目中提取出94个相互无依赖的子项目作为CryptoGuard的输入。针对Android，作者从google应用市场下载了6181个流行应用，涉及58个种类。作者使用Soot将这些app反编译为java字节码作为CryptoGuard的输入。

作者的测试环境是Intel Xeon(R) X5650 server (2.67GHz CPU and 32GB RAM)。对于Apache项目，平均运行时间是3.3分钟，中位数是1分钟。对于Android项目，分析时间超过10分钟的项目会被手动终止（552个，9%），平均分析时间是3.2分钟，中位数2.85分钟。

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%206.png](/images/2019-12-06/Untitled%206.png)

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%207.png](/images/2019-12-06/Untitled%207.png)

i) hard-coding default keys or passwords in the source code, ii) storing plaintext keys or passwords in configuration files, and iii) storing encrypted passwords in configuration files with decryption keys in plaintext in source code or configuration.

Apache项目中共有的不安全做法：1）在源码中硬编码默认密钥或者口令。2）在配置文件中存储密钥或者口令。3）在配置文件存储加密的口令，但是解密口令的密钥硬编码在代码中。

5个Apache项目做了假的hostname有效性检查(Spark, Ambari, Cxf, Ofbiz, Meecrowave)。6个Apache项目做了假的证书检查(信任任何证书)，包括Spark, Ambari, Cloudstack, Qpid-broker, Jclouds和 Ofbiz。默认不安全，可选安全的方式。

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%208.png](/images/2019-12-06/Untitled%208.png)

针对Android，作者发现95%的密码学误用来自第三方库。相比于Apache项目，Android项目中SSL/TLS和HTTP的误用比例更高。

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%209.png](/images/2019-12-06/Untitled%209.png)

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%2010.png](/images/2019-12-06/Untitled%2010.png)

## Limitations

**False Positive**

![CryptoGuard%20High%20Precision%20Detection%20of%20Cryptograp/Untitled%2011.png](/images/2019-12-06/Untitled%2011.png)

CryptoGuard只检测了误用的存在，并没有验证误用是否会在运行时触发

**False negatives due to refinements**

`byte[] key = DatatypeConverter.parseHexBinary("6A5B7C8A")`

以上代码会被认为是参数没有影响key，但实际上影响了。
