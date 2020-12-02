---
layout: post
title: "Pseudorandom Black Swans Cache Attacks on CTR_DRBG"
date: 2020-03-16 18:00:36 +0800
comments: true
categories: 
---

> 作者：Shaanan Cohney[1], Andrew Kwong[2], Shahar Paz[3], Daniel Genkin[2], Nadia Heninger[4], Eyal Ronen[5], Yuval Yarom[6]
> 
> 单位：[1]University of Pennsylvania, [2]University of Michigan, [3]Tel Aviv University, [4]University of California, [5]Tel Aviv University and COSIC (KU Leuven),  [6]University of Adelaide and Data61
> 
> 会议：41st IEEE Symposium on Security and Privacy (IEEE S&P 2020)
> 
> 原文：[https://eprint.iacr.org/2019/996.pdf](https://eprint.iacr.org/2019/996.pdf)

## ABSTRACT

CTR_DRBG是NIST推荐的随机数生成算法。作者发现流行的CTR-DRBG实现存在缺陷，会导致缓存侧信道攻击，使攻击者能够恢复CTR_DRBG的内部状态；结合实现中参数选择的缺陷，攻击者可以从TLS连接中恢复敏感数据。

作者在两种场景下验证了攻击的可行性：1）攻击者作为TLS连接服务端，利用缓存侧信道攻击恢复CTR_DRBG的内部状态，从而恢复受害者的TLS长期认证密钥。2）攻击者利用Intel SGX的特性进行blind attack，在不需要观察CTR-DRBG输出的前提下通过三次AES加密操作恢复CTR-DRBG的内部状态，并被动解密受害者的TLS连接数据

<!--more-->

## INTRODUCTION

现代密码学利用随机性来抵御攻击者对密码协议参与方的敏感信息的预测。随机数被用于密钥生成（公钥与对称），密钥交换（前向安全的DHE与ECDHE），签名（DSA, ECDSA的k）以及各种协议中防止重放攻击的nonce。

常见的伪随机数生成器（PseudoRandom number Generators, PRG）的实现，会从系统环境或者硬件中收集信息添加到熵池，再利用熵池生成seed，通过一系列算法将短的seed扩展成长序列。

在侧信道存在的情况下，PRG的安全性是未知的。CTR-DRBG是NIST SP 800-90A推荐的伪随机数生成器，但是此标准并没有考虑侧信道攻击的影响。目前还没有一个系统的研究针对侧信道存在下的PRG安全性的研究，本文作者首次进行了此方向的研究，并着手讨论以下问题：

- 现有的PRG设计是否会受到侧信道攻击的影响。
- 针对PRG的侧信道攻击会造成什么后果，攻击者会如何利用它们。

对于第一个问题，作者表示CTR-DRBG容易受到侧信道攻击的影响，因为一些流行的CTR-DRBG实现（OpenSSL 1.0.2 FIPS，NetBSD kernel，FortiOSv5 kernel，mbedTLS-SGX，nist_rng Library）的底层分组密码使用了T-Table AES算法，该算法不能抵抗侧信道攻击。

对于第二个问题，作者发现一些流行的CTR-DRBG在一些情况下不能正确的reseed，使预测攻击成为可能。对于OpenSSL，NetBSD和FortiOS，作者展示了将TLS连接作为目标并恢复受害者长期认证密钥的攻击。针对mbedTLS-SGX，作者展示了针对TLS连接的被动解密数据的攻击。

### 作者贡献

- 作者首次进行了针对CTR_DRBG在侧信道存在情况下的安全分析，展示了通过缓存侧信道进行的CTR-DRBG状态恢复攻击。
- 展示了在CTR-DRBG的reseed算法存在缺陷的情况下，结合上述的状态恢复攻击，通过让受害者连接恶意TLS服务器，对受害者的长期认证密钥进行恢复的攻击。
- 展示了一种新的差分分析技术，利用了运行在SGX enclave的CTR-DRBG的侧信道泄露，利用至少三条AES加密操作恢复CTR-DRBG内部状态。
- 展示了在上一步攻击的基础上，无需受害者连接恶意TLS服务器即可被动解密TLS连接数据的攻击。
- 爬取了NIST's Cryptographic Module Validation Program database，评估了CTR_DRBG的流行程度。

### Coordinated Disclosure

作者向OpenSSL，Fortinet和NetBSD报告了漏洞。OpenSSl回复说这些攻击不在其威胁模型之内。NetBSD和Fortinet修复了漏洞（CVE-2019-15703，DRBG insufficient entropy）。

## BACKGROUND

### Pseudorandom Generators

PRG又称为确定性随机比特Deterministic Random Bit Generators（DRBG），在输入相同时产生的输出相同。

PRG算法一般由以下四个函数组成：

- **instantiate:** 输入熵池取样I和nonce N，输出初始状态
- **generate:** 输入当前状态S，所需的比特长度nbits以及附加输入addin，输出随机序列和新状态S'
- **update:** 输入当前状态S，附加输入addin，输出新状态S'
- **reseed:** 输入当前状态S，熵池取样I和附加输入addin，输出新状态S'

PRG算法安全性：

- **backtracking resistance：**PRG的状态被攻击者获取后，攻击者仍然不能对之前的PRG输出进行区分
- **prediction resistance：**PRG的状态被攻击者获取后，攻击者仍然不能对之后的PRG输出进行区分

攻击者破坏PRG的能力：

- **get-output:** 攻击者可以调用generate，并且已知addin
- **get-state:** 攻击者可以获取当前状态S
- **set-state:** 攻击者可以选择S'并设置为PRG当前状态
- **next-ror:** PRG调用generate生成随机数，再选择另外一个同样长度的随机序列，并让攻击者对两者进行区分

作者定义的PRG with Input Security:

攻击者可以调用有限次的update/get-output/next-ror，并且在最后一次next-ror之前可以调用一次get-state，在这种前提下攻击者仍然无法区分PRG产生的随机数与任意随机序列。

### NIST SP 800-90 and Related Standards

包括四种算法，分别为CTR-DRBG, HMAC-DRBG, HASH-DRBG和DualEC-DRBG。其中DualEC-DRBG被认为有设计缺陷，导致潜在的后门。

四种PRG算法分别基于分组密码，HMAC，HASH和双椭圆曲线。

### AES

AES一般有11轮，可以分为四种操作：AddRoundKey, SubBytes, ShiftRow, MixColumns。

T-Table AES是针对AES的一种优化，可以通过查表的方式替换上述的后三种操作。并且AES最后一轮用到的查表与之前轮数的不同。下面是一个T-Table AES的参考实现，在加密时会用到5个表，解密时会用到六个表:

[https://github.com/openluopworld/aes_128/blob/master/lut/aes.c](https://github.com/openluopworld/aes_128/blob/master/lut/aes.c)

攻击者可以通过观察查表时的内存访问模式，进而推测受害者的AES密钥。

### Cache Attacks

缓存侧信道攻击可以利用访问某一内存的用时（rdtsc），判断某一地址的数据是否之前被读取过。有以下两类攻击方法：

- **Flush+Reload**

    攻击者首先将某一内存的数据从cache中清除（clflush），然后让受害者进程运行，然后攻击者reload之前的内存。如果受害者访问过这块内存，数据已经再次存入cache，reload的速度就会很快，否则会慢一些。通过这种方法攻击者可以定位受害者访问过的数据。需要注意的是，此方法需要攻击者与受害者处于同一地址空间或者拥有共享的内存页（shared libraries or kernel）；以及cache每次存入的数据是以cache line为单位，一般为64字节，需要攻击者通过其他的方式才能更精确的定位数据。

- **Prime+Probe**

    prime阶段攻击者通过大量的访问内存来填满cache，然后将控制权交给受害者。probe阶段攻击者再次访问第一步访问的内存，通过测量时间来判断cache miss，进而判断受害者的数据被加载到哪个cache line

![Pseudorandom%20Black%20Swans%20Cache%20Attacks%20on%20CTR_DRBG/Untitled.png](/images/2020-03-16/Untitled.png)

### CTR-DRBG

CTR-DRBG通过利用分组密码算法对一个递增的计数器进行加密来产生输出，分组密码算法可以是3DES(64bits)，AES(128, 192, 256bits).

![Pseudorandom%20Black%20Swans%20Cache%20Attacks%20on%20CTR_DRBG/Untitled%201.png](/images/2020-03-16/Untitled%201.png)

![Pseudorandom%20Black%20Swans%20Cache%20Attacks%20on%20CTR_DRBG/Untitled%202.png](/images/2020-03-16/Untitled%202.png)

K和V的长度分别为分组密码密钥和分组的长度，seed的长度和nonce的长度相同，均为keylen+blocklen。

PRG实例化时利用熵池样本I和任意的nonce N生成一个临时变量t，再令K=V=0, addin=t，调用update生成初始状态 S =  (K, V, c=1)，c为reseed counter。

状态更新和产生随机数算法如上图所示。如果攻击者可以获取算法2里11-14行的K和V，并且可以猜测addin的值，攻击者就可以预测PRG之后的输出。

另外一个需要注意的地方是，update只有在所需nbits的数据生成之后才会调用，如果攻击者一次请求很长的随机数输出，中间过程中K是不会改变的。

reseed会以当前状态S，熵池样本I和addin作为输入，返回一个新状态并重置reseed counter。

**Attacking CTR DRBG**

1. 假设攻击者能够获取PRG输出，并且可以获取算法2里11-14行的密钥K，攻击者就可以计算V。
2. 攻击者通过猜解addin，并利用算法1计算新状态
3. 攻击者再次猜解addin，并通过算法2预测随机数输出

上述攻击的关键是攻击者如何获取PRG输出，如何获取K以及如何猜解addin。

作者表示，很多实现的addin均采用低熵或者可预测的数据作为addin（例如时间戳），可以在2^{21}的复杂度内猜解两个addin。

## STATE RECOVERY ATTACK

### 设计缺陷

- **FortiOSv5**

    FortiOSv5将linux kernel默认的/dev/urandom实现替换成nist_rng的CTR-DRBG实现，该实现底层的分组密码算法使用了T-Table AES。

    并且FortiOS的CTR-DRBG在update时没有加入addin，也没有使用reseed。

- **NetBSD**

    NEtBSD与FortiOS具有同样的问题。但是在generate的时候会用rdtsc作为addin。在2^{30}次调用generate后，CTR-DRBG会reseed。reseed counter会在每次调用generate后加1，而不是计算每一个分组加1，所以攻击者可以每次调用generate生成一个很长的随机数，此过程K不会改变。

- **OpenSSL 1.0.2 FIPS Module**

    此版本的CTR-DRBG同样使用了T-Table AES。每次调用generate后，会使用微秒级的时间，一个计数器以及PID来update。在调用2^{24}次generate后，进行reseed。

    OpenSSL 1.1版本的CTR-DRBG使用了AESNI指令，所以不会被作者的攻击影响。

### CACHE ATTACK DETAILS

T-Table AES是最容易受到缓存侧信道攻击的算法之一。作者的目标是AES加密的最后一轮，这一轮用的查表和其他轮数的不同。作者假设攻击者可以获取加密的输出，并且可以通过缓存侧信道攻击得到T[q_i]的信息，那么就可以用以下等式计算最后一轮的轮密钥。

$$c_i = T[q_i]\bigoplus k_i\\ k_i = T[q_i] \bigoplus c_i$$

![Pseudorandom%20Black%20Swans%20Cache%20Attacks%20on%20CTR_DRBG/Untitled%203.png](/images/2020-03-16/Untitled%203.png)

**Handling Missing Information**

在真实的攻击场景中，攻击者不可能精确的获得i和q_i的信息。因为攻击者只能获得被害者在AES最后一轮整体的访问内存的信息，所以访问的顺序无从得知；另外，由于cache中是按cache line存储数据，所以每一次查表都会存入cache 64个字节的数据（作者发现SBOX表里的每一个字节会被连续存储四次），攻击者需要从16个字节里区分哪个是被害者访问的数据。

根据统计特性，AES的最后一轮平均会访问11个cache line的数据，所以最终密钥的每个字节的可能性为11*16=176种。

为了准确恢复密钥，作者采取了以下方法：

![Pseudorandom%20Black%20Swans%20Cache%20Attacks%20on%20CTR_DRBG/Untitled%204.png](/images/2020-03-16/Untitled%204.png)

![Pseudorandom%20Black%20Swans%20Cache%20Attacks%20on%20CTR_DRBG/Untitled%205.png](/images/2020-03-16/Untitled%205.png)

攻击者通过一次generate生成一个很长的随机序列（这一过程中K不变，每一个分组V+1），在每一次AES加密的最后一轮进行缓存侧信道攻击，获取trace data。在S中计算密钥每一个字节值的次数，取最大值即为密钥。

**Obtaining Trace Data**

- 作者利用ticker（指令缓存探针，通过cache hit判断受害者代码执行的位置）来保证对SBOX的cache攻击的时机。作者在加密函数的开头和结尾设置两个ticker，但任一个被触发后，作者对SBOX进行cache攻击。
- CPU会尝试学习程序的cache访问模式，并预先将数据存入cache（prefetching）。这对作者的侧信道攻击影响很大。作者通过以不规则的方式存放cache line来影响CPU的预测。
- 如果针对SBOX的cache攻击时间过长，攻击者就无法准确的获取trace。作者的方法是通过不断刷新受害者程序的指令cache，进而减缓加密操作的速度。在作者的实验中，可以将加密操作的时间从2微秒增加到34微秒。
- 由于加密操作是使用相同的key对一个递增的计数器加密，所以作者可以用推测出的密钥解密数据并验证是否为一递增的数列来判断是否恢复正确的密钥

### Evaluation

**攻击场景**

作者假设攻击者可以在目标机器上执行非特权代码。被害者进程在同一机器上通过调用CTR_DRBG的generate产生一个2KB的随机输出。攻击者利用Flush+Reload攻击监控generate内部AES加密时的cache访问，并用上述的攻击手段恢复PRG的状态。

**软件与硬件环境**

OpenSSL 1.0.2, 用nist_rng库的AES128为PRG的底层密码算法（T-Table），无AESNI指令。

Intel i7-3770 Quad Core CPU, with 8GB of RAM and 8MB last level cache.

**结果**

禁用缓存预取的情况下，作者可以以4.58%的误报率恢复PRG状态。允许缓存预取的情况下，结果如右图，成功率为12%，误报率为28.5%。状态恢复攻击平均用时19秒

![Pseudorandom%20Black%20Swans%20Cache%20Attacks%20on%20CTR_DRBG/Untitled%206.png](/images/2020-03-16/Untitled%206.png)

## ATTACKING TLS

在PRG状态恢复攻击的基础上，作者进一步针对TLS进行了攻击，目标是获取受害者的长期认证密钥。

### Background

- 在TLS1.3之前，RSA作为TLS在密钥交换阶段使用的公钥加密算法。RSA使用时需要进行填充，以防止选择明文攻击。PKCS#1 v1.5是常用的填充格式。
- PKCS#1 v1.5填充之后的长度与RSA公钥的n长度相同，中间的Padding部分是非零的随机数。
- ECDSA是TLS中标准的公钥签名算法，相关计算公式如右图，其中d为私钥，G为生成元，Q为公钥。k为随机数，(r, s)为签名。
- TLS相关
    - 由客户端发起TLS连接，发送支持的密码套件；服务器选择密码套件
    - 客户端在验证服务端身份后，将PMS利用RSA填充加密发送给服务端
    - 需要双向认证时，客户端需要提供ECDSA证书以及签名。
    - 双方根据PMS以及其他参数计算对称加密使用的密钥。

![Pseudorandom%20Black%20Swans%20Cache%20Attacks%20on%20CTR_DRBG/Untitled%207.png](/images/2020-03-16/Untitled%207.png)

$$Q= dG\\r=kG\\s=k^{-1}(H(m)+rd)\\d=(sk-H(m))r^{-1}$$

### Attack Overview

作者的PRG状态恢复攻击需要2000字节连续的的随机数输出，在TLS中，当用于密钥交换的RSA的n长度大于等于2048字节时，PKCS#1 v1.5的填充格式满足随机数的要求（2048-48-3=1997）。

作者假设受害者作为客户端连接到一个恶意的攻击者控制的TLS服务端（<1.3），并且使用RSA进行密钥交换以及ECDSA作为公钥签名算法。作者还假设攻击者可以在受害者的机器上执行非特权代码。

作者针对TLS的攻击流程如下：

1. **受害者连接到攻击者控制的TLS服务器**

    作者假设受害者浏览的网页包含攻击者控制的脚本，恶意脚本会初始化与攻击者控制服务器的TLS连接，服务器使用RSA进行密钥交换，并且要求双向认证。

2. **恢复PRG状态**

    客户端使用服务端RSA公钥加密PMS，并进行PKCS#1填充，攻击者在此时进行cache攻击。

3. **ECDSA签名生成**

    受害者发送自己的ECDSA证书，并用随机生成的nonce对CertiﬁcateVerify信息进行签名

4. **恢复受害者生成的nonce**

    攻击者可以离线搜索PRG生成nonce时的输入熵和addin数据。对于恢复的nonce，作者可以通过重新计算签名的r来验证

5. **恢复密钥**

    利用第四步的nonce，作者可以恢复受害者的ECDSA私钥，并冒充受害者的身份。

### Using PKCS#1 v1.5 in OpenSSL 1.0.2 for Nonce Recovery

上述流程里最重要的是第四步，作者先介绍了OpenSSL进行PKCS#1填充以及ECDSA nonce的各个步骤：

- 在调用CTR-DRBG的generate生成PKCS#1填充时（在作者的实验中，填充长度为1996），generate内部会调用两次update，一次在利用分组密码生成随机数之前，另一次在生成随机数之后。作者的cache attack在第二次调用update之前进行。
- PKCS#1填充只能使用非0值，如果第一步生成的随机序列中含有0x00，则会对于每一个0x00重新调用generate生成一字节非0值，每个字节会导致两次update。
- OpenSSL进行ECDSA签名时，会调用RAND_seed来对PRG进行reseed，TLS握手数据的SHA256哈希会用作熵。
- OpenSSL会调用一次RAND_add，秒级的时间会被用作熵值。
- 最后OpenSSL调用CTR-DRBG的generate生成nonce，在生成nonce之前会先调用一次update。

攻击者可以通过PKCS#1填充获取1996字节的随机数，在此期间受害者会进行125次AES加密，即攻击者最多可以收集125次cache attack的trace。

在整个攻击的过程中，作者主要遇到并解决了如下问题：

1. 生成随机数作为PKCS#1填充时，存在字节为0x00时，会重新调用generate替换对应字节，使攻击难度增加

    作者提出，由于随机字节的分布是均匀的，所以1996个字节有(255/256)^{1996}=0.000405的概率不出现0x00，也就是平均每2470次TLS握手会出现一次1996个字节不出现0x00。结合之前状态恢复攻击的成功率，作者表示每2^18次TLS握手作者可以成功恢复一次PRG状态。

2. 在PKCS#1填充随机数之后与nonce生成前，PRG会reseed，攻击者必须获取RAND_seed 和RAND_add的参数才能进一步攻击
    - FortiOS没有实现RAND_seed和RAND_add，而是依赖于nist_drbg内部的reseed counter，因此调用RAND_seed和RAND_add不会造成update；除此之外，FortiOS在generate时没有使用addin，进一步降低了攻击的复杂性。
    - OpenSSL 1.0.2 FIPS模块也没有在RAND_seed和RAND_add调用时对CTR-DRBG进行reseed，而是将熵值添加到熵池里，在之后reseed时被使用。作者发现添加的熵值仅有12位。
    - NetBSD会用rdtsc的低32位输出作为addin。

### Evaluation

作者可以直接对FortiOS实现的ECDSA的nonce进行恢复，因为它没有使用addin。对于OpenSSL FIPS，作者可以在半个小时内完成恢复。

作者表示，利用CPU可以以每小时2^{22}次进行搜索，利用GPU可以达到每小时2^{35}次。

![Pseudorandom%20Black%20Swans%20Cache%20Attacks%20on%20CTR_DRBG/Untitled%208.png](/images/2020-03-16/Untitled%208.png)

## ATTACKING FULL ENTROPY IMPLEMENTATIONS

作者针对Intel SGX实现了一种状态恢复攻击，不需要观察PRG输出，也不需要耗费大量资源求解熵值。

但是，为了达到这个目的，作者需要攻击者具有更强的侧信道攻击能力，以更高的分辨率观察cache的访问模式。

作者假设受害者的进程运行在攻击者控制的操作系统的主机的SGX enclave里。

### **Threat Model**

SGX是Intel针对x86架构的扩展，可以保证enclave内的数据不会被enclave外的实体读取（包括kernel或者hypervisor）。理论上可以通过SGX保护用户级的代码和数据，免受恶意操作系统或者hypervisor的高权限攻击者攻击。

作者假设一个拥有root权限的攻击者控制了整个OS，这符合Intel SGX的威胁模型，通过enclave保证数据的机密性和完整性。与之前不同的是，作者不需要限制受害者连接到恶意服务器或者使用有缺陷的PRG。

### **Differential Cryptanalysis of CTR DRBG**

通过cache attack可以获得每一次T-Table查表的16个候选对象。

作者的差分分析可以利用两次加密的cache trace来确定精确地从16个候选对象中定位正确的字节。

利用了相邻两次AES加密的明文相差1，在第0轮可以进行差分分析恢复一个字节。

利用恢复的一个字节，进一步在第1轮恢复4个字节。

最后利用恢复的4个字节，在第2轮恢复16个字节。

### **Fine Grained Cache Attack**

为了获得差分攻击所需的trace，拥有root权限的攻击者需要通过Prime+Probe来监控cache访问模式。

作者利用了controlled-channel攻击获得了细粒度的时间分辨率，controlled-channel attack利用了禁用enclave页表的present bit达到攻击目的。

通过将T-Table的页面设为not-present，受害者此时访问T-Table时会发生enclave exit，从而将控制权转移到攻击者控制的OS。

由于所有的T-Table位于同一页面（刚好4K），所以攻击者需要在每一次T-Table访问之间进行controlled-channel attack。

作者选择了包含栈顶帧的页面进行controlled-channel attack，因为每次利用T-Table查表，进程会首先从栈上读取查表的索引。每次在这里将控制权返回给OS，攻击者就可以利用Last Level Cache Prime+Probe攻击来获取受害者查表对应的cache line。

一个最大的问题是，栈地址被enclave loader随机化了。作者的解决方法如下：

1. 通过controlled-channel attack，在受害者程序进入AES加密函数之前将控制权还给OS
2. 在此时，将enclave的除了包含代码的所有页面标记为not-present，由于进入函数的第一条指令是push rbp，向栈顶写数据的操作触发了segfault，OS此时就可以定位enclave栈顶所在的页面。

为了降低攻击的噪声，作者利用Intel的缓存分配技术将攻击者和受害者的进程划分到同一块LLC上，并且利用isolcpus 内核引导参数将它们隔离到同一个CPU核上。

### Evaluation

**实验环境**

Lenovo P50 laptop equipped with 16 GB of RAM and an Intel i7-6820HQ CPU clocked at 2.7GHZ with a 8MB L3 cache，Ubuntu 16.04

受害者为基于mbedtls-SGX的HTTPS客户端

**攻击场景**

受害者通过HTTPS访问[www.cia.gov](http://www.cia.gov/)，客户端的所有密码学函数都在enclave里运行。

攻击者首先通过Prime+Probe攻击恢复CTR-DRBG的状态，该CTR-DRBG用于生成前向安全的ECDH私钥（五次对一个递增的计数器进行的AES256加密操作）。

利用恢复的ECDH私钥，攻击者可以计算后续的premaster key以及对称加密密钥，进而解密受害者TLS连接的流量。

**结果**

作者在1000次测试中，有36%的测试成功恢复了enclave内CTR-DRBG的内部状态。online阶段，利用LLC Prime+Probe进行攻击用时2s；离线阶段，恢复CTR-DRBG状态和计算TLS后续数据用时可以不计。

## 个人评价

~~作者整篇论文的出发点其实就是现有的CTR-DRBG存在缺陷，然后增加了各种千奇百怪的脑洞，给了攻击者各种各样的能力最终完成了攻击。~~

本质上没有很新的东西，就是将现有的针对AES的侧信道攻击转移到了应用AES的随机数产生器上。