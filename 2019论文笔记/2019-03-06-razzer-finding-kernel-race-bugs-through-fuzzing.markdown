---
layout: post
title: "Razzer: Finding Kernel Race Bugs through Fuzzing"
date: 2019-03-06 19:06:27 +0800
comments: true
categories: 
---
作者：Dae R. Jeong, Kyungtae Kim, Basavesh Shivakumar, Byoungyoung Lee, Insik Shin

单位：KAIST, Purdue University, Seoul National University

出处：IEEE Symposium on Security and Privacy 2019

资料：[Paper](https://csdl.computer.org/csdl/proceedings/sp/2019/6660/00/666000a279.pdf) | [Github](https://github.com/compsec-snu/razzer)

<hr/>
## 1. Abstract & Introduction

作者在本文中提出了自己设计的用来发现内核中的数据竞争类型
漏洞的工具Razzer。

Razzer结合了两种技术：
1. 静态分析找到潜在的存在竞争的代码。
3. 确定性的线程交错技术

作者针对Linux内核实现了一个Razzer的原型。并发现了30个新的内核中的Race bug，其中16个已经被确认且被修补。

在内核中，数据竞争通常是很多有害行为的根源。数据竞争可能会导致内存污染（例如缓冲区溢出、UAF），最后导致权限提升。例如CVE-2016-8655, CVE-2017-2636, and CVE-2017-17712。

作者最后也将Razzer和其他Fuzzing工具进行对比，并总结了对比的结果。


<!--more-->
## 2. Problem Formulation

目标程序内两条内存访问的指令执行，满足下面的3个条件，即可认为是数据竞争：
1. 访问相同的内存区域
2. 至少其中一个指令为写指令
3. 两个指令并发执行

引入如下标注：
- $RacePair_{cand}$：可能满足上面三个条件的RacePair
    - $RacePair_{true}$：已确定满足上面三个条件的RacePair，是$RacePair_{cand}$的子集
        - $RacePair_{benign}$：属于预期内的数据竞争
        - $RacePair_{harm}$：非预期的数据竞争

Razzer的一个重要的设计目标是防止任意的假阳性结果。为了实现这个目标，作者提出了下面两设计时的要求：
- **R1**: 找到一个执行$RacePair_{cand}$的输入程序，也就是说需要得到一个多线程对用户态程序，这个程序的每个线程能够在内核态分别执行到$RacePair_{cand}$的每条指令。
- **R2**: 找到一个能够并发执行$RacePair_{cand}$的程序。

单单只有**R1**没办法保证$RacePair_{cand}$被同时执行了，还需要找到一个特定的能够穿插执行的线程。

现在的大部分工具都是专注于满足这两个条件中的一个条件。
        
### 2.1 Race Example: CVE-2017-2636

如下图所示是CVE-2017-2636的成因图，n_hdlc->tbuf是一个指针，$RP^A$的432行被push到free_list（之后会被free），在440行处置零，$RP^B$的216行将tbuf取出，并在后面也将其push进free_list，可以看到，如果出现图中的执行顺序，就会导致Double-free的产生。

![](/images/2019-03-06/media/15451401057106/15451452427458.jpg)

### Requirement Study: Traditional Fuzz Testing

传统的Fuzzing，例如AFL是用来Fuzz用户态的程序，Syzkaller则是用来Fuzz内核。这两种技术都是专注于寻找能够提高Coverage的输入（R1）。由于没有满足R2，因此发现数据竞争的效率是不高的。

### Requirement Study: Thread Interleaving Tools

Random thread interleaving tools (SKI or the PCT Algorithm)，这些都是专注于实现R2这个要求。SKI一开始的时候就随机化地选择一个内核现场，然后执行它，直到遇见内存访问指令。停在这个地方后，再随机选择一个内核现场进行执行，一直重复这个过程。

## 3. 设计

Razzer同时使用了静态分析和动态分析的方法。首先Razzer通过静态分析获得潜在的存在数据竞争的位置，然后进行分两阶段的动态分析。第一阶段是单线程Fuzz，目标是找到一个能执行到数据竞争位置的用户空间输入程序（满足R1）。第二个阶段是多线程Fuzz，在这个阶段，要构建多线程的程序，通过自己定制的hypervisor进行控制，实现确定性的线程交错（为了满足R2）。一旦找到了一个竞争，Razzer就会输出具体的用户态输入程序，以及触发的根源信息。

### 3.1 静态分析

静态分析的目标是发现内核中所有的$RacePair_{cand}$，每个$RacePair_{cand}$由两条内存访问指令组成。通常来说，用Points-to分析会是一个比较好的选择，然而Points-to在准确性和性能上都有缺陷。在准确性上，Points-to有很高的假阳性概率。性能上，时间复杂度是$O(n^3)$，n代表被分析程序的大小。

Razzer用两个方法解决了这些问题：
1. 在准确性上，Razzer遵循了Points-to的方法，对于假阳性的结果，通过后面的动态分析过滤，。
2. 在性能上，Razzer对Kernel进行了分部分的分析，它将内核根据模块进行分，并对每个模块进行预分析。在Linux Kernel这种情况下，Points-to可以直接根据源代码的目录结构进行分，Linux源码每一个子目录都代表了一个模块了。

### 3.2 动态分析

Razzer将目标kernel跑着一个虚拟化环境里，以保证确定性的vCPU运行。Razzer修改了一个现有的hypervisor，修改后的hypervisor具有下面功能：
1. 可以为每个虚拟CPU核心设置断点。Razzer为内核提供了一个新的Hypercall（hcall_set_bp）接口，这样就可以在Guest kernel里为某个CPU设置专门的断点了。hcall_set_bp有两个参数，vCPU_ID指示了需要设置断点的CPU，guest_addr指示了断点的地址。这里还有一个问题是，在hypervisor上运行的kernel，kernel上面又运行着不止目标程序的其他程序，可能有其他的程序导致内核触发了断点，作者在这里的处理方法上用VMI来判断当前的内核线程上下文，获取内核线程id进行进一步判断，是否是目标程序。
2. 可以恢复执行。在两个内核线程都停在了对应的断点地址后，Razzer会让vCPU重新开始执行，这样就会并发的执行$RacePair_{cand}$了。这时还会有顺序问题，因为两个指令的不同的执行顺序决定了是否会发生错误的行为（通过hcall_set_order实现），Razzer也提供了一个Hypercall用来设定执行的顺序。
3. 确定竞争是否真的发生。当断点同时被触发了，hypervisor会检查两条指令访问的目标地址是否一样，如果一样则认为$RacePair_{true}$。

如下图所示是Hypervisor的工作原理

![](/images/2019-03-06/media/15451401057106/15517582400996.jpg)

### 3.3 Fuzzing

这一步将前面的技术整合起来，实现Fuzzing。Razzer的Fuzzing过程分为两步：
1. 单线程Fuzzing。找出一个单线程的用户态程序，可以触发$RacePair_{cand}$
2. 多线程Fuzzing。找出一个多线程的用户态程序，可以触发恶性数据竞争（基于第一步得到的结果）

整个过程如下图所示：

![-w1189](/images/2019-03-06/media/15451401057106/15517147538925.jpg)


**单线程Fuzzing**。单线程Generator首先初始化生成$P_{st}$，这是一个单线程的用户态程序，有一系列随机的系统调用（生成的方法是用的Syzkaller预先定义的系统调用语法）。然后通过单线程Executor运行这些$P_{st}$，并测试这些$P_{st}$的执行是否覆盖了$RacePair_{cand}$。如果执行路径覆盖到了，就会进行标注，然后将这个$P_{st}$传到下一步的多线程进行处理。判断完一轮后，再对$P_{st}$进行变换（插入或者删掉一些系统调用），重新开始。

**多线程Fuzzing**。这一步多线程Generator将单线程Fuzzing输出的一些$P_{st}$作为输入，然后输出$P_{mt}$，$P_{mt}$是$P_{st}$的多线程版本。同时用hypercall插桩，如下图所示是将$P_{mt}$转化为$P_{st}$。Executor执行每一个$P_{mt}$，如果判断有Race，则将$RacePair_{cand}$认定为是$RacePair_{true}$。这时再对$P_{mt}$进行修改，传回到Generator继续新的一轮。如果$P_{mt}$最终触发了崩溃，就认为这是一个$RacePair_{harm}$

![-w424](/images/2019-03-06/media/15451401057106/15517146976591.jpg)

这里还有一个细节是在两个线程最后都加入了一个随机的syscall，为了确保恶性数据竞争出现后程序会崩溃。

## 4. 实现

Razzer修改了一些已有的框架进行实现：
1. 静态分析：SVF，增加修改了638行的C++代码
2. Hypervisor：QEMU，增加修改了652行C代码
3. Fuzzer：Syzkaller，增加修改6403行的Go代码和286行的C++代码

## 5. 评估

实验环境：
- Intel(R) Xeon(R) CPU E5-4655 v4 @ 2.50GHz (30MB缓存) ，512GB内存
- Ubuntu 16.04 / Linux 4.15.12 64-bit

用修改后的KVM/QEMU创建了32个VM，16个VM用于单线程，16个VM用于多线程Fuzzing。为了与Syzkaller进行比较，作者也用QEMU为Syzkaller创建了32个虚拟机，与Razzer使用相同的计算资源。

Razzer不对要分析的目标内核进行修改，对于每个内核版本，就做两步处理：
1. 用LLVM的编译套件生成Bitcode，进行静态分析
2. 然后用GCC进行编译，得到最终的内核二进制文件。

作者从三个方面对Razzer进行了评估：
1. 新发现的数据竞争漏洞
2. 分析了Razzer静态分析的效率
3. 测量了Hypervisor的性能开销，并与最新的Fuzzing工具进行比较

### 5.1 新发现的漏洞

作者测试的Linux内核，版本包括了从v4.16-rc3（2018.02.25发布）到v4.18-rc3（2018.6.1发布）。总共跑了大约7个星期。下图是作者最终发现的漏洞，总共发行了30个恶性的数据竞争，在报告了后，其中16个已经被确认，这里面的14个已经提交了patch。作者在这里特别强调了一下，Linux被很多很多不同的工程师和研究者在Fuzz，其中也包括Google的Syzkaller团队在他们的云上一直在跑他们的fuzzer（为了更早地发现漏洞），可想而知在计算资源不是很强的情况下，要发现新的漏洞是很不容易的。

![](/images/2019-03-06/media/15451401057106/15516880215819.jpg)

如下图所示是Razzer发现漏洞的效率。

![](/images/2019-03-06/media/15451401057106/15516884023920.jpg)

Razzer所发现的一部分漏洞有很严重的安全影响，一些可以被攻击者用来进行权限提升。这其中还有一些漏洞都是在内核里存在了很久的古老漏洞了，例如refcount_dec是从Linux v2.6.20（2007）引入的漏洞，以及KASAN: slab-out-of-bounds write in tty_insert_flip_string_fixed_flag在Linux v2.6.38引入（2011）。

Razzer还为发现的漏洞生成包含漏洞成因的报告，为开发者的漏洞修复过程提供了帮助。

### 5.2 静态分析的效率

如下图所示是Razzer静态分析在每个模块上花的时间

![](/images/2019-03-06/media/15451401057106/15517474512206.jpg)

### 5.3 Hypervisor开销

因为Razzer用了hypercall来确定化vCPU的行为，所以需要测试这所带来的额外开销，如下是作者实验得到的hypercall带来的时间开销

![](/images/2019-03-06/media/15451401057106/15517481637838.jpg)

### 5.4 Fuzzing效率的对比

与Syzkaller相比，吞吐量上表现比Syzkaller要差（因为用了），但是Razzer在发现恶性数据竞争的速度是比Syzkaller快的（至少23-86倍）

与SKI进行对比，由于SKI的源码没办法获得，作者修改了Razzer，模拟SKI，也是显示了Razzer的效率比SKI高很多，因为Razzer的搜索空间比较小，直接在$RacePair_{cand}$里进行测试，SKI很多做的是无用功。