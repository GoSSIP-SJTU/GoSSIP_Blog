---
layout: post
title: "IMIX: In-Process Memory Isolation EXtension"
date: 2018-12-25 20:44:46 +0800
comments: true
categories: 
---

作者：Tommaso Frassetto, Patrick Jauernig, Christopher Liebchen, Ahmad-Reza Sadeghi

单位：Technische Universität Darmstadt, Germany

出处：USENIX Security 2018

资料：[Paper](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-frassetto.pdf) | [Slide](https://www.usenix.org/sites/default/files/conference/protected-files/security18_slides_jauernig.pdf)

---

## 1. Abstract & Introduction 

现在，研究者们已经提出了很多复杂的攻击技术，例如（JIT/blink）ROP、Counterfeit object-oriented programming。这些技术都能够使得攻击者达到任意代码执行、数据攻击的目的。与此同时，研究者们也提出了很多针对代码重用攻击的的防御方案，例如Control-flow integrity（CFI），Code-pointer integrity（CPI）。但是这些防御方案都依赖着较强的Memory isolation的保证，例如CFI的Shadow stack，CPI的Safe-region，以及一些随机化的秘密，都是需要进行隔离，不能被攻击者随意读写。由此可见进程内的内存隔离的重要性。

另外，进程内的内存隔离和进程间的内存隔离的区别从下图可以看出：
![](/images/2018-12-25/media/15450554637231/15457061192579.jpg)

本文中，作者为Intel的x86 CPU设计了一个轻量级扩展IMIX，IMIX可以做到进程内的内存隔离。作者采取的办法是扩展x86的指令集架构，增加一个新的内存访问的权限，用来标识内存页是否是安全敏感。这些内存页只能被新设计的指令来访问。与之前的工作不同的是，IMIX并不是一个完整的防御方案，而是作为Primitive来使用。

<!--more-->
## 2. Adversary Model

Memory corruption：作者假设已经存在了一个Memory corruption的漏洞，通过这个漏洞，攻击者能够重复地进行数据的读写。

Sandbox code execution：攻击者可以正常执行已经受保护的应用程序的代码，但是攻击者只能通过Memory corruption漏洞来影响应用程序。同时应用程序采用了CPI或者CFI以及代码随机化这样的技术来抵御任意代码执行。然而攻击者可以尝试通过Memory corruption漏洞来绕过这些缓解措施，并且假设要绕过缓解措施，需要破坏缓解措施中的metadata。

## 3. IMIX设计

如下图所示是IMIX的架构。
![](/images/2018-12-25/media/15450554637231/15451937048773.jpg)
缓解措施的正常工作依赖于其本身的数据和代码的完整性。由于像CPI和CFI这样的缓解措施，需要将他们的metadata都保存在被保护进程本身的内存空间里，以避免过高的性能开销。因此攻击者可能会利用应用程序本身的Memory corruption漏洞来破坏metadata，从而绕过缓解措施。

传统的方法里，防御方案开发者通过实施W ⊕ X或Execute-only memory来保证代码的完整性，而数据完整性就依赖于其他的进程内内存隔离来保证。然而现有的内存隔离技术，例如Instrumentation和Data hiding，都需要使用者在高性能开销和安全性之间进行抉择。

作者设计的IMIX提供了高效率，安全性，基于硬件的进程内内存隔离方案。那些属于缓解措施本身的metadata会被分配到*isolated pages*，然后标上一个特殊的访问权限。只能用作者提出的`smov`指令来进行数据访问。除了`smov`指令，IMIX还包括内核扩展以及编译器支持。

### 3.1 Hardware

`smov`指令重用了`mov`指令的逻辑。
修改了内存访问逻辑，下面情况会触发fault：
1. 除了`smov`以外的对隔离内存的访问
2. `smov`被用来访问普通的内存

也就是说，访问普通内存用`mov`，访问保护内存用`smov`。这里需要注意的是为什么`smov`不能访问普通的内存，可以考虑在CPI方案下的这种攻击：攻击者修改了某个指针，这个指针本身应该指向Safe region，但是现在攻击者把他指到自己可控的一个buffer（在普通内存里），这时候假设程序已经采取了IMIX，然后程序尝试用`smov`去解引用指针，就是不允许的了，也就是能成功防御住这种攻击。

### 3.2 Kernel

内核态通过Page tables来进行虚拟内存和物理内存的映射，每个页在Page table里被描述为PTE（Page table entry），PTE里包含了一些metadata，metadata里描述了一些关于这个页的访问权限。用户空间的程序可以通过系统调用来请求修改这个页的访问权限。

作者在内核上的扩展就是提供了一个额外的IMIX保护的访问权限，

### 3.3 Compiler

IMIX提供了两个上层Primitive：
1. 分配保护内存
2. 访问保护内存

这些内存保护的Primitive既可以用在缓解措施上，也可以用在敏感数据的保护上。IMIX为这两种用途都提供了优化过的接口，例如：
1. 像CPI这样的缓解措施是作为LLVM的pass来实现的，工作在IR层。IMIX提供了一些IR primitive用来进行IR的修改
2. 对于应用程序开发者来说，IMIX提供了源码标注：特殊标注的变量会被分配在保护内存里，所有的对这个被保护的变量的访问会通过`smov`实现

## 4. Implementation

### 4.1 CPU Extension

作者将IMIX Protection flag放在PTE的第52个bit，因为这个bit没有被CPU保留，通常是被MMU无视的。CPU需要更新一下来支持IMIX的访问策略，非`smov`指令只能访问普通页，`smov`指令只能访问隔离页，否则CPU就会抛出一个fault。这个逻辑的实现需要对下x86-64指令集架构进行修改。作者使用了一个硬件模拟器来展示他们的设计的。

作者想要选择Wind River Simics（全系统模拟器）来进行IMIX的模拟。但是Simics这个模拟器比较慢，没办法启动Linux内核。因此作者使用了Intel Simulation and Analysis Engine（SAE）add-on。之后的文字里就通过SAE来简单代表这两者的结合。SAE支持模拟完全的x86系统。对于CPU，内存，以及相关的硬件例如MMU的插桩，作者是使用*ztools*来实现。*ztools*是用C/C++实现的一个共享库。

为了对模拟的系统进行插桩，*ztools*为一些特殊的hook注册了回调。作者首先通过注册Initialization hook的回调来确认*ztool*以及被初始化成功。然后注册一个回调，当新的指令被添加到CPU指令cache时就会执行。如果有`mov`或者`smov`访问内存的指令出现了，就会再注册一个指令替换的回调。这个注册的回调能够替换指令，或者选择执行原来的指令。在这个回调handler里，作者实现了IMIX的访问逻辑。
1. 检查指令所访问内存的的IMIX Protection flag
2. 如果是普通的指令尝试访问普通的内存，就会继续执行原来的指令，以防止指令Cache的修改；如果是`smov`指令尝试访问隔离页，就首先将指令从指令cache里去掉，然后执行*ztool*实现的指令。其他情况就会抛出一个fault。

### 4.2 Operating System Extension

操作系统需要做的事是为隔离页在PTE里标记出IMIX Protection bit。作者修改了Ubuntu 16.04 LTS发行版的默认内核（当时是4.10）。进程可以通过mprotect系统调用来请求内核对页进行标记（用PROT_IMIX这个flag）。这里需要注意，页一旦被标记为PROT_IMIX，要想去掉这个flag，就只能unmmap这段内存。

### 4.3 Compiler Extension

作者修改了LLVM 5.0（选择LLVM而不是GCC的原因是大部分的Memory-corruption防御方案用的都是LLVM），并且移植到了LLVM 3.3（CPI用的是LLVM 3.3），这个修改主要是在IR上，为CPI提供`smov`指令，以及x86的后端，用来生成机器码。另外，还设计了一个attribute，可以用来隔离单个变量，例如保护Cryptographic secret。

对IR的扩展：IR通常提供两种内存访问指令，即`load`和`store`，`load`指令从内存读数据到Temporary register，`store`指令用来把Temporary register的数据存到内存里。因此作者设计了两个IMIX对应的指令，即`sload`和`sstore` .

Attribute：Data-only attack通常很难防御，为了给开发者在源码级别有效的方法去防护敏感的数据例如Cryptographic keys，作者设计了一个IMIX attribute，可以在编程时通过对变量进行标注，从而使得其分配在隔离页。所有的队标注出来的变量的访问都会使用IMIX IR的指令。

对x86后端的修改：处理`sload`和`sstore`两个IR指令。

### 4.4 Case Study: CPI

作者将CPI进行了移植，使用上IMIX。CPI使用了Safe region特性来保证代码指针的完整性，以及抵御代码重用攻击。所有的代码指针，指向指针的指针等等，都会被移到Safe region，这样Memory corruption漏洞就没办法覆盖到它们了。返回地址会用Shadow stack来保护。CPI的安全性依赖于保护x86-64的Safe region不被修改，CPI将Safe region放在一个随机的地址处，并且将地址保存在Segment里（通过段寄存器%gs来获取）。在编译时，CPI的pass会将所有的代码指针和额外的关于边界的metadata移到Safe region。已经有研究表明CPI的实现是脆弱的，因为Safe region的位置是是可以爆破的。

作者将Data hinding的工作用IMIX来实现：
1. 修改CPI的内存分配函数，不仅仅是分配Safe region，还需要设置IMIX Protection bit
2. 修改Compiler runtime，提供对Safe region的访问（`smov`）。

## 5. Security Analysis

攻击者想要访问被保护数据，就需要用`smov`来进行访问，因此作者只能依赖于代码注入或重用可信代码，或者操作数据流，或者访问IMIX的配置接口

### 5.1 Attacks on Trusted Cocde

IMIX假设代码注入攻击，代码重用可以用现有的缓解措施例如W⊕X、CFI、CPI来防御。这样攻击者就不能注入`smov`指令或者是访问重用可信代码了。

### 5.2 Attacks on Data Flow

数据流攻击通常是很难防御的，可信代码需要保证输入的数据是来源于由IMIX保护的数据，或者在使用数据前先进行过滤。

### 5.3 Attacks on Configuration

一个绕过缓解措施的普遍的访问就是把缓解措施关掉，例如想要绕过W⊕X，可以通过代码调用mprotect系统调用将数据段标为可执行。

攻击者有两种方法重新配置IMIX：
1. 利用操作系统的接口修改内存访问权限
2. 直接操作PTE

对于第一种方法，作者假设攻击者能修改mprotect系统调用的参数，但是由于接口已经保证IMIX页是不能取消IMIX标志位的，除非unmmap，所以是不可行的。

对于第二种方法，假设攻击者在内核里有Memory corruption的漏洞，作者认为内核也同样可以使用IMIX来进行保护，例如隔离页表。

## 6. Evaluation

实验环境：Ubuntu 14.04 LTS，Linux Kernel 3.19.0，Intel Core i7-6700 CPU、32GB RAM

测试数据集：SPEC CPU2006 benchmark suite

作者将IMIX的实现移植到了CPI上来隔离Safe region。

IMIX在性能上表现地相比MPK、MPX要出色。这里由于硬件是用模拟器模拟的，因此在跑数据集的时候很慢，作者就将smov里的指令都替换成了mov指令，在计算机上直接跑。

![](/images/2018-12-25/media/15450554637231/15452043747961.jpg)

## 7. Related Work

如下是各个相关技术的一些特征的对比。
- Policy-based Isolation指的是内存保护本身是没办法通过任意内存读写绕过的。
- Fast Interleaved Access指的是频繁的访问隔离内存和非隔离内存不会带来额外的开销。
- Fails Safe指普通内存访问指令不能访问保护内存

![](/images/2018-12-25/media/15450554637231/15457062855474.jpg)


