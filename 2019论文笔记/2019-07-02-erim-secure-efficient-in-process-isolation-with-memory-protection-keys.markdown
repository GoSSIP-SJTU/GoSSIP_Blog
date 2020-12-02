---
layout: post
title: "ERIM: Secure, Efficient In-process Isolation with Memory Protection Keys"
date: 2019-07-02 18:53:53 +0800
comments: true
categories: 
---
作者：Anjo Vahldiek-Oberwagner, Eslam Elnikety, Nuno O. Duarte, Peter Druschel, Deepak Garg

单位：Max Planck Institute for Software Systems (MPI-SWS), Saarland Informatics Campus

出处：USENIX Security 2019

资料：[Paper](https://people.mpi-sws.org/~druschel/publications/erim.pdf)
<hr/>
## 1. Introduction & Background

隔离敏感数据可以提高许多应用程序的安全性和健壮性。可以保护像Heartbleed案例中密码学密钥不被泄露，保护程序语言的运行时。

在这篇文章里作者介绍了ERIM，可以提供基于MPK的开销比较小的隔离方案，即使权限切换很频繁（每秒100000次切换），ERIM的平均开销仍然是小于1%的。主要的思想是运用Intel MPK机制。ERIM可以很容易应用到新的和已经存在的应用程序，不需要编译器的支持，可以在主干的Linux内核上运行。
<!--more-->
### 1.1 Intel MPK

每个进程都会有16个Protection Key，最多能将进程的地址空间分为16个域。*PKRU*存储了每个Key的具体访问权限。

当要改变某个域的访问权限时，只要通过一个用户态的指令*WRPKRU*（11-260 cycles）即可，不需要系统调用，不需要对页表修改，刷新TLB等。

由于$WRPKRU$是用户态的指令，ERIM需要保证恶意的攻击者没办法利用这个指令来切换权限。

### 1.2 Linux Support for MPK

Linux从4.6版本开始支持MPK，页表的值会有4位用来存储Protection Key。有专门的系统调用（pkey_mprotect）可以为某个页绑定Protection Key。

## 2. Design

- $T$: 可信部分代码
- $U$: 剩余的不可信部分代码
- $M_T$: 需要保护的内存
- $M_U$: 剩下的不需要保护的内存

ERIM所采取的措施：
1. 当$U$在执行时，不允许其访问$M_T$
2. 想要访问$M_T$，需要将程序跳转到$T$的入口点，在$T$的出口处再撤销访问$M_T$ 的权限

### 2.1 High-level Design Overview

将$M_T$和$M_U$映射为两种不同的域，保证$M_U$总是允许访问，$M_T$是永远不能被$U$访问。但是$U$可以通过跳转到$T$的*call gates*来获得访问权限。

ERIM要解决的一个难点是防止$U$利用*WRPKRU*指 问权限。ERIM依赖于二进制检查（Binary Inspection）来保证只有安全的*WRPKRU*出现在可执行页里。

安全的*WRPKRU*需要满足如下条件：
(A) $T$的入口
(B) 在*WRPKRU*的后面会马上检查指令设置的参数是预期的参数。

在x86上，为了防止*WRPKRU*出现在长指令的中间，或者是两条指令的中间，作者通过二进制重写，重写所有的包含*WRPKRU*为不包含其的等效的指令。这个可以在编译器中静态实现，也可以通过二进制检查完成。

### 2.2 Threat Model

ERIM不对$U$做任何假设，$U$可以做任何事情，甚至可能包含*Memory corruption*和*Control-flow hijack*的漏洞。

ERIM假设$T$的Binary不含漏洞，不会泄露$M_T$的数据。

### 2.3 Call Gates

在$T$的开头通过*WRPKRU*更新对$M_T$的访问权限，在函数执行完后再通过*WRPKRU*撤销对$M_T$的访问权限。
![-w333](/images/2019-07-02/media/15445160181965/15446674937217.jpg)

### 2.4 Binary inspection

二进制检查由检查函数和监听（Interception）两部分组成

#### 2.4.1 Inspection Function

二进制检查需要扫描一系列的可执行页是否出现*WRPKRU*指令或者*XRSTOR*指令。找到后，就会检查是否安全的。

检查条件1：ERIM需要开发者维护一系列的$T$的入口点

检查条件2：就简单的去验证指令后面是否跟着用于检查权限的指令即可

#### 2.4.2 Interception

通过配置Seccomp-bpf Filter来监听mmap、mprotect、pkey_mprotect等系统调用是否将页映射为可执行权限。Seccomp没有直接的方法来监听*PKRU*，作者是通过SECCOMP_RET_TRACE来通知Tracer。Tracer会允许来自$T$调用的这类系统调用，对于来自$U$的就会拒绝。不过ptrace的方式开销很高，但是将页映射为可执行权限这样的操作一般比较少。但是如果真的很多，也可以通过在写一个简单的Linux内核模块来实现。

有了这样的监听的存在，$U$必须要进入$T$才能映射可执行的页。

### 2.5 Lifecycle of an ERIM Process

ERIM在程序main函数执行之前进行$M_T$的初始化，同时也初始化默认的MPK域，给$M_U$使用。ERIM会修改内存分配函数，内存分配函数通过*PKRU*来判断当前是处于$T$还是$U$，然后选择分配不同的内存。

### 2.6 Developing ERIM Applications

作者设计了三种程序开发模型。

1. Binary-only模式。通过一个动态库，Stub funciton的形式封装入口点（用LD_PRELOAD的方式加载这个库）。作者用这种模式来隔离SQLite和Node.js
2. Source模式。这种模式需要在源码里插入Call Gates。作者用这种模式实现隔离OpenSSl里的密码函数和会话密钥
3. Compiler模式。修改编译器，让编译器在编译时插入Call Gates。作者用这种模式实现CPi

Source示例模式如下所示。

![-w522](/images/2019-07-02/media/15445160181965/15615552902592.jpg)

## 3. Rewriting Program Binaries

针对X86的重写策略，以*WRPKRU*为例

1. 如果这个指令跨越了多条指令，则比较简单，在这些指令之间插入NOP即可
2. 如果是在一条长指令的里面，则需要替换为语义相等的指令，或者一系列指令，

对于可以用源码重新编译的情况，重写可以在编译器里完成。如果不能，就要在运行时做二进制重写。

## 4. 实验

隔离Web服务器里的密码学密钥。例如Heartbleed这个案例。long-term key不是很频繁地被访问，基本上每个会话就几次。Session key则会更频繁地被范文，每秒超过$10^6$次（NGINX这种高吞吐量的Web服务器）。ERIM可以高效地隔离Session key。作者修改Openssl的libcrypto库，隔离了Session key和基本的密码相关代码。

隔离Native库。例如Java和Javascript VM通常都依赖于第三方的Native库来保证性能，ERIM可以用来隔离这些运行时。

隔离CPI的Safe Region。作者修改了编译器里CPI/CPS的部分。

## 5. 评估

作者在Linux上实现了两个版本的ERIM原型。一个版本是一个77行代码的Linux内核模块，用于监听所有的mmap和mprotect调用，防止$U$映射可执行页以及防止$U$修改二进制检查句柄函数。另一个版本是基于ptrace实现。另外还有内存分配函数的修改，以及二进制重写的实现。

作者用Microbenchmark和上面提到的三个软件来评估ERIM。

### 5.1 Microbenchmarks

如下图所示，测试了Switch的性能。

![-w304](/images/2019-07-02/media/15445160181965/15615579237254.jpg)

### 5.2 Protecting Session Key in NGINX

作者配置了NGINX仅仅使用ECDHE-RSA-AES128-GCM-SHA256这个密码学套件，以及会话的AES加密。

下图是吞吐量的实验结果。

![-w376](/images/2019-07-02/media/15445160181965/15615581264529.jpg)

### 5.3 Isolating managed runtimes

用ERIM隔离程序语言的运行时。下图显示了开销的实验结果。

![-w349](/images/2019-07-02/media/15445160181965/15615583080857.jpg)


### 5.4 Protecting sensitive data in CPI/CPS

用ERIM来隔离CPI的Safe Regin以及CPS。下图展示了开销的实验结果

![-w349](/images/2019-07-02/media/15445160181965/15615585477273.jpg)

### 5.5 与现有技术的比较

可以看到ERIM的运行时性能还是比较好的，切换用的都是*WRPKRU*指令。

![-w338](/images/2019-07-02/media/15445160181965/15615586722808.jpg)


## 6. 个人评价

*WRPKRU*指令在glibc的pkey_set函数也有，如果攻击者劫持控制流后通过ROP调用这个函数，也还是能修改访问权限。对于这种情况，ERIM是否考虑到了？这种情况要解决的话，应该可以直接把glibc里的pkey_set这个函数给删掉，也没有什么函数是依赖这个函数的。