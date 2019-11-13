---
layout: post
title: "Analyzing Semantic Correctness with Symbolic Execution: A Case Study on PKCS#1 v1.5 Signature Verification"
date: 2019-04-21 15:58:12 +0800
comments: true
categories: 
---

题目：Analyzing Semantic Correctness with Symbolic Execution: A Case Study on PKCS#1 v1.5 Signature Verification

作者：Sze Yiu Chau[1], Moosa Yahyazadehy[2], Omar Chowdhuryy[2], Aniket Kate[1], Ninghui Li[1]

单位：[1]Purdue University， [2]The University of Iowa 

出处： The Network and Distributed System Security Symposium (NDSS) 2019

原文：[https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_04A-4_Chau_paper.pdf](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_04A-4_Chau_paper.pdf)

/images/2019-04-21：[会议:NDSS2019](https://www.ndss-symposium.org/ndss-program/ndss-symposium-2019-program/)，  [个人主页](https://www.cs.purdue.edu/homes/schau/)， [PPT](https://www.ndss-symposium.org/wp-content/uploads/ndss2019_04A-4_Chau_slides.pdf)

<hr/>
## 一、背景

密码协议是保证数据安全可靠的协议，在现代的各个领域中都具有不可代替的作用。但是，开发一个安全可靠的密码协议并不是一件容易的事情，把密码理论应用到实际的生产中需要经历漫长的开发过程，在这之中，即使有一点点的疏忽，也可能导致原来安全的密码协议变得不再安全。

来源于数学理论的密码结构，在假定的前提下，通常是很难破解的，而在实际中，这些密码结构通常需要使用一种中间结构来表示（Glue Protocol，本文中是指 PKCS#1 v1.5），以适应不同的输入数据和数据格式。

当密码协议设计好之后，就要按照设计说明书来实现它，以此来保证密码协议的安全性，倘若在实现过程中没有严格按照标准实现，就有可能达不到原来设定的安全等级，使得密码协议容易遭到各种攻击。

PKCS#1 v1.5 就是一种中间结构，在 RSA 算法中，PKCS#1 v1.5 定义了 RSA 的数学表示形式，在某些特殊的参数设定下（例如：使用较小的公钥指数 e），PKCS#1 v1.5 签名验证方案可以让攻击者在没有私钥、也不对模数进行分解的情况下伪造一个数字签名。

像 IPSec 中的密钥生成函数 ```ipsec_rsasigkey``` ，所生成的公钥指数 e = 3，而且没有任何选项可供选择（比如生成更大的公钥指数），再结合代码实现中对签名数据的处理不当（例如：在 Openswan 中），就使得签名信息容易被攻击者伪造，造成不必要的损失。因此，检查代码实现中的语义信息的正确性就变得一项重要的工作。

<!--more-->
## **二、提出的方法以及解决的问题**

由于签名验证算法实现的语义正确性得不到很好的保证，导致签名信息被攻击者伪造这种错误，一般需要手工去审计代码的语义正确性，但这样就需要花费大量的人力去做检查。因此，作者就开发了一个自动化的检测工具，用来分析密码协议的实现中存在的语义正确性（Semantic Correctness）问题。在这篇文章中，作者只针对了 PKCS#1 v1.5 签名验证方案，检测该方案实现中是否存在语义正确性问题，代码是否遵循预先定义的规范。


## **三、技术方法**

为了识别出密码协议中存在的语义缺陷（Semantic Weakness），作者使用的是现成的符号执行工具：[**KLEE**](https://github.com/klee/klee)，但是直接使用该工具检测密码协议，可能会存在一些扩展性问题，因为密码协议的输入通常是一些结构化的数据，只是某些字段可能是不定长的。

由于密码协议的数学结构通常是非线性的，因此，在符号执行过程中，约束求解器通常很难分析它，而在 PKCS#1 v1.5 签名验证方案中，各个可变长度的字段之间通常与输入存在一种线性的关系，如图 1 所示，拥有这种线性的约束关系，就可以让符号执行引擎自动的生成很多有意义的测试例子（Concolic Test Case），这种技术，在文中称作：Meta-Level Search。

![图 1 签名验签的输入输出结构](/images/2019-04-21/图1-IO结构.png)

为了进一步解决符号执行的扩展性问题，作者还使用了一种 [Human-In-The-Loop](https://arxiv.org/pdf/1805.03450) 技术，根据协议设计和输入数据格式的相关信息，把输入进行一个粗粒度的划分（把输入结构中的某些字段设置为符号值：Symbolic，剩下的字段设置为具体值：Concrete），用于指导符号执行，使得求解器更容易求解所得的符号表达式（相当于原始表达式的子表达式）。因此，可以避免由于循环或者递归导致的符号爆炸等问题。

当符号执行引擎找到了一个目标结果之后（代码实现和最初的规范存在偏差），还需要快速的定位到产生问题的代码所在的位置，因此，为了辅助问题分析，作者开发了一个约束起源追踪器 CPT（Constraint Provenance Tracking），用于从符号引擎输出的结果（Path Constraint）中定位源代码所在的位置。

### **1. 符号执行**

对于复杂的输入，符号执行引擎一般难于分析，并且非常耗时。因此，在本文中，作者对输入数据进行一个粗粒度的划分，并结合专业知识，把部分符号值具体化，简化符号执行引擎的负担，具体分析如下：

首先，作者在这篇文章中分析的目标是：使用 PKCS#1 v1.5 方案的 RSA 签名验证过程，表 1 中列出了本文中所涉及的一些符号表示：

![表 1 本文中所使用到的符号表示](/images/2019-04-21/表1.png)

签名的输入 *I* 和验签的输出 O，如图 1 所示，它们都具有统一的结构，从一个 0x00 开始，然后是 ```块类型``` ：BT (Block Type)，接着是 ```填充字节``` ：PB (Padding Bytes)，文档要求该字段的长度大于等于 8 个字节，并且每个字节都为 0xFF，填充的规则是使得整个 I/O 结构的长度是 |n|，填充结束后使用 0x00 表示，最后再跟上一个 DER 编码的 ASN.1 结构：AS。AS 的结构如图 1 所示（在这个图中假设哈希算法使用的是 SHA-1），是一种树状表示形式，每一项都是一个 ```{Tag, Length, Value}``` 这样的一个三元组，Length 表示 Value 的字节长度，父节点的 Length 字段表示它所有子节点的数据长度之和。因此，该结构存在明显的线性关系，利用这种线性的关系，可以很好地辅助符号分析引擎分析代码。

在图 1 中，u = 5, y = 0, x = 2 + u + 2 + y, z = 20, w = 2 + x + 2 + z。 |AS| + 3 + |PB| = |n|。

正是由于这种明显的线性关系，使得输入空间可以被分解成不同的、更小的子问题，并且每一个子问题都能够很好地被符号执行引擎分析，对于每一个子问题，作者再利用 Meta-Level Search 技术去自动的生成 Concolic Test Case。在文中，作者把输入空间分解为三个部分，并把它叫做： Test-Harnesses （TH1、 TH2、 TH3），如下表所示。在每一个 Test-Harness 中，都像正常应用程序一样调用 PKCS#1 v1.5 的签名验证函数，以检查程序是否能够正确的处理不同长度的变量。此外，在符号执行过程中，对于循环的处理也可以比较好的解决，因为 PKCS#1 v1.5 方案中可以根据 |n| 的值来确定循环的边界。

| Test Harness | 目的                     | 说明                                                          |
| ------------- | ------------------------ | ----------------------------------------------------------- |
| TH1           | Checking of BT, PB, z, y | PB 可变长度（Symbolic），但 u,w,x,y,z 保持 Concrete           |
| TH2           | Checking the OID         | 同上                                                        |
| TH3           | 跟 TH1、TH2 对比          | u,w,x,y,z 保持 Symbolic，其它保持 Concrete（与 TH1/TH2 相反） |

由于 u、w、x、y、z、|PB| 和 |n|/8 之间存在线性关系，因此，在 TH1 和 TH2 的符号执行过程中，可以构造出很多有意义的输入，用于测试程序的语义正确性。在 TH3 中则刚好和 TH1、TH2 相反，TH1 中为 Symbolic 的变量，在 TH3 中变成 Concrete， 在 TH1 中为 Concrete 的变量，在 TH3 中变成 Symbolic。

### **2. CPT**

由于现代的符号执行引擎（KLEE）的执行过程比较耗时，它们通常会简化和重写路径约束，以此来减少约束求解的时间开销。因此，作者就开发了一个追踪工具：CPT （Constraint Provenance Tracking），用来辅助分析符号执行引擎所产生的结果（符号表达式），以便于更好的从符号表达式中定位源代码所在的位置。该工具只在 KLEE 中添加了 750 行代码，并且对 KLEE 的时间和性能开销几乎可以忽略不计。


### **3. 签名验签过程**

对于输入的消息 m，根据所选定的哈希算法（例如 SHA-1），计算消息的哈希值 H(m)，并把哈希结果和其它的一些参数信息（Mete-Data）打包到一个 DER 编码的 ASN.1 结构中（图 1 中的 AS 部分）作为 AS，然后再加上一些填充信息和类型信息等，构成签名的输入 *I*（图 1 中的 I/O），所生成的输入 *I* 的长度是 |n|，最后计算签名 S：

$$
 S = I^d mod (n)
$$

当验签程序收到签名对象之后（一般是 X.509 证书），就从签名对象中提取出 S，然后再计算出 O：

$$
 O = S^e mod (n)
$$

验签程序使用指定的哈希算法重新计算消息的哈希值，并把计算结果和 O 中的哈希结果（或者整个 I/O 对象）进行比较，判断签名是否合法。

在这里有两种比较方法：


(1). **基于构造的验签（Construction-Based Verification）**：验签程序计算出消息的哈希值之后，重新构造一个 *I'*，并把 *I'* 和原来的 *I* 进行比较，相等则说明原来的签名正确，否则不正确。

(2). **基于解析的验签（Parsing-Based Verification）**：验签程序计算出消息的哈希值之后，直接和 O 中的哈希值进行比较，相等则说明原来的签名正确，否则不正确。


在这篇文章中，作者只关注第二种比较方法（因为第一种他攻击不了），因为这种方法更容易遭受攻击，而太过仁慈（Too Lenient）的解析程序给攻击者提供伪造签名的空间，也既所谓的 [Bleichenbacher](https://download.hrz.tu-darmstadt.de/pub/FB20/Dekanat/Publikationen/CDC/sigflaw.pdf) 低指数暴力破解攻击。

### **4. 签名伪造**

前文中已经提到，签名的输入和验签的输出具有相同的格式，如下所示：
```
    0x00  ||  0x01  || PB  || 0x00  || AS
```
由于对 PB 的检查太过松散，或者是没有检查，因此，就给攻击者提供了攻击的可能，攻击者可以构造如下形式的签名 O'：
```
    0x00  ||  0x01  || GARBAGE  || 0x00  || AS
```
文章中针对低指数的签名进行攻击（e = 3），并且当前设定的哈希算法是 SHA-1（其它哈希算法也可以），假设伪造的签名为 S'：
$$
 S' = (k_{1} + k_{2})
$$
则验证程序就会计算 O'：
$$
 O' = S'^3 = (k_{1} + k_{2})^3 = k_{1}^3 + 3k_{1}^2k_{2} + 3k_{2}^2k_{1}  + k_{2}^3
$$
k1 和 k2 的选择需要满足如下三条：

    (1). k1 立方 的高位必须和 PB 之前的数据一样，既： 0x00 0x01。
    (2). k2 立方 的低位必须和 PB 之后的数据一样，既： 0x00 || AS。
    (3). k2 立方 的高位、k1 立方 的低位、3倍 k1 平方 乘以 k2 的值、3倍 k2 平方 乘以 k1 的值，保持在 PB 当中。

**查找 k1：** 

最简单的方法就是求立方根，例如，当 |n| = 1024 的时候，PB 前面是 15 个 0 和 一个 1，既：
$$
    0x00\space0x01\space0x00\space0x00\space......\space0x00 = 2^{1008}
$$
既：
$$
 k_{1} = 2^{336}
$$
这只是针对 |n| = 1024 的情况，更通用的方法请阅读论文。

**查找 k2：** 

直接查找 k2 很难，因此，作者给出了另一种找 k2 的方法：
$$
 令 ： S'' = (0x00 || AS), n'' = 2^{|S''|}, |S''| = |AS| + 8
$$
只要 k2 满足：
$$
 k_{2}^e \equiv S'' mod (n'')
$$
既：
$$
 \phi(n'') =  \phi(2^{|S''|}) = 2^{|S''|-1}
$$
再根据扩展欧几里得算法来求得 e 的逆 f ，使得：
$$
 e \cdot f \equiv 1 \space mod (2^{|S''|-1})
$$
 最后的 k2 就是：
$$
 k_{2} =  S''^{f} \space mod (n'')
$$

## **四、实验评估**

**实验环境：**

- CPU：i7 3740QM
- 内存：32GB DDR3
- OS： Ubuntu 16.04

### **1. 测试工具的可行性**

作者使用两个已知 Bug 的软件进行测试（分别为 GnuTLS 1.4.2 和 OpenSSL 0.9.7h），来展示工具的可行性，如表 2 所示，Lines Changed 表示需要对源代码进行少量的修改之后才能进行测试。对于每一个 Test-Harness，如果 Accepting Path 大于 1，则表示该代码的实现极有可能出现问题（等于 1 也不一定是没有问题），既：实现上和规范有偏差。为了检验可能出问题的代码，作者使用差分交叉验证（Differential Cross-Validation）的方式来检验检测结果。

在这两个版本的库中，它们都是用 ```基于解析的验签``` 方式来实现验签功能，

对于 OpenSSL 0.9.7h，TH1 和 TH2 所产生的 Accepting Path 是因为它的验签程序都接受 AS 后面跟随有多余数据的情形，这也是它出现 Vulnerability 的原因（当 e = 3 的时候），而在 TH3 中，4 个 Accepting Path 是因为它除了接受 ASN.1 的 y = 0， z = 20 这种正常的情况之外，还接受 y = 128， z = 128，这都是由于它的 ASN.1 解析器对数据检查不够严格导致的。

对于 GnuTLS 1.4.2，TH1 产生的 3 个 Accepting Path 是因为它接受过短的 PB，而可以把填充字节隐藏到（挪到）ASN.1 的 Algorithm Parameter 中，这个错误也可以造成低指数的签名被伪造；TH2 中出现 21 个 Accepting Path 是因为它对 ASN.1 中的 OID 的检查太过松散导致，TH3 虽然只有 1 个 Accepting Path，但是作者在 KLEE 中依然发现了一个由解析器导致的内存错误漏洞。

![表2 使用两个已知bug的软件进行测试](//images/2019-04-21/表2.png)

### **2. 对 15 个开源库进行测试**

![表3 使用两个已知bug的软件进行测试](//images/2019-04-21/表3.png)

如表 3 所示，表中阴影部分表示代码实现没有问题（通过了交叉验证），带星号的库表示在测试的过程中把，该库对 PCKS#1 v1.5 的解析有内部实现方式和调用其它库实现的方式，而在文中把它配置成使用它本身实现的方式。

作者认为 **基于构造的验签** 方式不会出现问题，因此，把它当做交叉验证的基本准则，并把 GnuTLS 3 .5.12 作为交叉验证的基准，因为它使用基于构造的验签方式，并且所产生的路径最少。

**Openswan 2.6.50：**
- Ignoring padding bytes，(CVE-2018-15836)，代码片如下：
  
![Snippet1](/images/2019-04-21/Snippet1.png)
    
    已成功开发出攻击脚本（e = 3，H() = SHA-1，|n| = 1024/2048）。

**StrongSwan 5.6.3:**
- Not checking algorithm parameter (CVE-2018-16152)

    已成功开发出攻击脚本（e = 3，H() = SHA-1，|n| = 1024，需要 2^21 次迭代）。

- Accepting trailing bytes after OID (CVE-2018-16151)

    已成功开发出攻击脚本（e = 3，H() = SHA-1，|n| = 1024，迭代次数与上一个同数量级）。

- Accepting less than 8 bytes of padding

    结合 Not checking algorithm parameter 缺陷，已成功开发出攻击脚本（e = 3，H() = SHA-1，|n| >= 1024）。

- Lax ASN.1 length checks

    结合 Accepting less than 8 bytes of padding 缺陷，已成功开发出攻击脚本（e = 3，H() = SHA-1，|n| >= 1024）。

**axTLS 2.1.3：**
- Accepting trailing bytes (CVE-2018-16150)

    已成功开发出攻击脚本（e = 3，H() = SHA-1，|n| = 1024，需要 2^17 次迭代，几分钟就可以完成）。

- Ignoring prefix bytes，代码片如下：
  
![Snippet2](/images/2019-04-21/Snippet2.png)

    只是说容易攻击，但并没有表明是否已经攻击成功（或者开发出了攻击脚本）。

- Ignoring ASN.1 metadata (CVE-2018-16253):

![Snippet3](/images/2019-04-21/Snippet3.png)

    已成功开发出攻击脚本（e = 3，H() = SHA-1，|n| >= 1024）。

- Trusting declared lengths (CVE-2018-16149)

    作者通过构造一个伪造的签名，把 ASN.1 结构中的 z 值设置为一个非法值（过大的值），导致非法内存访问，是的验证程序崩溃，从而导致 DOS 攻击。

**MatrixSSL 3.9.1：**
- Lax ASN.1 length checks
- Mishandling Algorithm OID，代码片如下（psAssert 宏在 Release 版本中将不复存在）：

![Snippet4](/images/2019-04-21/Snippet4.png)

**GnuTLS 3.5.12：**
- 暂无

**Dropbear SSH 2017.75：**
- 暂无

**libtomcrypt 1.16：**
- Accepting trailing bytes

    当 e 足够小， |n| 足够大的时候可以攻击（作者没说是否攻击成功）。

- Accepting less than 8 bytes of padding，代码如下：

![Snippet5](/images/2019-04-21/Snippet5.png)

    结合 Accepting trailing bytes 缺陷，攻击成功（e = 3，H() = SHA-1，|n| = 1024）。

- Lax AlgorithmIdentifier length check

**mbedTLS 2.4.2：**
- Lax algorithm parameter length check
  
    ![Snippet6](/images/2019-04-21/Snippet6.png)

**BoringSSL 3112, BearSSL 0.4 and wolfSSL 3.11.0：**
- 暂无

**OpenSSL 1.0.2l and LibreSSL 2.5.4：**
- 暂无

**PuTTY 0.7：**
- 暂无

**OpenSSH 7.7：**
- 暂无



## **五、优缺点**


### **优点：**

- 相比于人工检查，工作量确实减少了很多，对于较大规模的测试比较有利。
- 这种只针对某些特定的结构进行分析的方法，可以比较方便的应用到别的数据结构上。


### **缺点：**

- 作者对目标程序进行测试之前，一般需要对代码进行少量的修改才能开始测试（比如代码中的循环等）。
- 基于源码的检测工具，只能检测开源代码中产生的漏洞，对于不开源的代码，则无能为力。
- 对于交叉验证描述不太清楚，而且叙述也比较少，基本上没看明白它是怎么实现交叉验证的。
- 代码尚未开源。


## **六、总结和看法**

作者在这篇文章中通过使用符号执行的方式来检测密码协议库中的语义正确性问题，并且针对的目标是 PKCS#1 v1.5 这种特定格式的签名验证方案。此外，为了使得符号执行更加简单、高效，作者把输入数据进行了一个粗粒度的划分（划分为：TH1、TH2 和 TH3），使用控制变量法把某些变量设置为具体值（Concrete），以此来减少约束求解器的求解难度，并能够通过这种方法来生成更多有意义的 Concolic Test Case，最后通过作者自主开发的 CPT 工具，可以快速方便的定位出现问题的代码所在的位置。通过使用两个已知漏洞的库做测试，验证了作者的这套工具的可行性，然后再对 15 个当前的密码库进行测试，发现 6 个库中出现语义缺陷问题，导致签名可以被攻击者伪造，甚至是产生 DOS 攻击等。最后作者针对这些缺陷，还开发了相应的攻击脚本，并且成功实施了攻击，最终产生 6 个新的 CVE。

整篇文章叙述性的东西比较多，没有一个总体的逻辑框图（比如，Overview 之类的），并且实现部分也说的比较少，基本上整篇文章都在讲述那 15 个密码库所产生的问题，以及 2 个用来做可行性分析的库，最后给了一部分实施攻击的理论模型，并阐述了几个攻击场景，以及所对应的 CVE，总体感觉就是：都在描述实验。

