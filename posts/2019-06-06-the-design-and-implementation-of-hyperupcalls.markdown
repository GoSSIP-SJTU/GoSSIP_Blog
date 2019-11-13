---
layout: post
title: " The Design and Implementation of Hyperupcalls"
date: 2019-06-06 11:04:39 +0800
comments: true
categories: 
---
作者：[Nadav Amit](https://research.vmware.com/researchers/nadav-amit ), [Michael Wei](https://www.linkedin.com/in/miwei/ )

单位：VMware Research  

出处：USENIX Annul Technical Conference 18  (CCF-A, 计算机体系结构/并行与分布计算/存储系统)

原文：[The Design and Implementation of Hyperupcalls](https://www.usenix.org/system/files/conference/atc18/atc18-amit.pdf )  

<hr/>

## 简介  
硬件虚拟化引入了抽象的虚拟机，使得宿主机（Hypervisor）可以运行多个客户机（Guest）操作系统。每个客户机都会认为自己运行在独立的硬件之上。虚拟化的目标就是明确区分虚拟机和真机，这带来一个语义鸿沟，就是虚拟和真机不知道对方在特定场景下做了什么选择。

解决这样的语义鸿沟往往会带来严重的性能问题，前沿的做法是**半虚拟化**。半虚拟化引入*hypercall*和*upcall*。其中前者像是Guest向Hypervisor引起的系统调用，后者反之。这种设计有很多问题：

- 半虚拟化机制需要做上下文切换，如果频繁地使用的话，会带来性能损耗（may be substantial ）
- 切换不能并行
- 开发一个半虚拟化特性需要guest和hypervisor配合开发

另一类方案是**虚拟机自省**，这种方案的实践高度依赖于操作系统的结构，操作系统的一些微小变化可能就令其失效，甚至引入新的攻击面。最后，这类方案只能简单沦入入侵检测领域。

基于上面的问题，作者提出了用以Hypervisor和Guest通信的Hypercall这一类机制，该机制有如下特点：

- 高效
- 容易部署
- 功能全面

为了证明上面三个点，作者实现了TLB Shootdown、Discarding Free Memory、Tracing、Kernel Self-Protection这4种Hyperupcall。（事实上，Hyperupcall是一个Hypervisor开发者提供给操作系统开发者的一个界面，方便操作系统开发者实现自己的某些需求）

每种Hyperupcall都通过基本框架进行部署，在基本框架中，作者实现了一个编译器和一个验证器，它们对Hyperupcall做复杂性和有限性的验证，以保证Hyperupcall不会太复杂和越过限制来执行。

<!--more-->
## 设计

### 总览

![figure1](/images/2019-06-06/hu-figure1.png)

一个Hyperupcall从编写到被调用的过程如上图。

首先，一个Hyperupcall被操作系统开发者用C语言编写，然后经过基本框架的编译和验证，得到eBPF字节码；在操作系统引导时，这些eBPF字节码被注册到Hypervisor中；注册后的eBPF字节码经过Hypervisor的eBPF验证后，由eBPF JIT编译器编译成JIT代码；当Guest OS引起一个Event的时候，JIT代码在Hypervisor中被执行。

JIT代码可以在Hypervisor中直接访问Guest的内存。

### 构建Hyperupcalls

Hyperupcall被调用在一些Event发生之时，客户机操作系统的开发者可以为自己想监控的Event注册Hyperupcall，并编写Event发生时Hypervisor要做的事情。

![table2](/images/2019-06-06/hu-table2.png)

#### 使用eBPF保证代码安全

在本工作的安全模型中，Guest是untrusted的，因此Hypervisor要能够验证Guest要注册的代码。

作者选用eBPF虚拟机作为Hyperupcall的载体，这个选择出于以下考量：

- BPF比较成熟，已经有很多应用。eBPF又扩展了BPF的功能
- eBPF有可证明验证的特性
- eBPF有现成的LLVM后端，方便以C语言开发为主的操作系统开发者使用

这里作者选择eBPF，所淘汰的选项是SFI、proof-carrying code和安全语言（如：Rust）

#### 从C到eBPF-基本框架

基本框架满足三个特性：

- 框架要能保证系统符号到客户机地址的翻译

- 解决eBPF的一些局限
- 框架定义一个简单的向Hyperupcall传数据的界面

对于客户操作系统的符号，框架保证了客户机操作系统从符号到偏移的正确翻译。

eBPF对于本工作的局限主要在于，不支持循环；指令集不包括原子操作；不能使用自修改代码、函数指针、静态变量、汇编代码、不能太长。为了解决这些局限，作者提供了一组Helper函数，如下：

![table3](/images/2019-06-06/hu-table3.png)

对于传数据的界面，就是在Hyperupcall被调用时，Guest向Hypervisor传哪些数据，如下：

![table4](/images/2019-06-06/hu-table4.png)

（这个表反映了hyperupcall对Hypervisor和Guest双方的访问能力）

### 编译

这一节主要讲作者对eBPF LLVM后端进行的修改。

#### Guest内存访问

Guest内存被当作数据包访问，访问数据包的形式为DPA，为了实现这个，作者解除了LLVM后端中数据包大小必须小于64K的限制。

#### 地址翻译

在Hypervisor里，对于要访问的GVA，有一段直接对应的HVA，Hyperupcall直接访问HVA就可以访问到客户机内存。作者扩展了LLVM后端，使用*address space*这个成员来实现上述功能。

#### 边界检查

作者在编译器中（？）添加了访存前检查边界的代码。

#### 上下文Caching

上下文指针会被频繁访问，作者将其的Caching机制实现在了编译器里，而不是让它作为参数传过来。

### 注册

客户机随时可以注册，但是大多数都是注册在引导的时候，客户机在注册时需要提供：

- Hyperupcall event ID
- Memory registration
- Hyperupcall bytecode

其中，对于Global Hyperupcall，Memory registration不能大于虚拟机所需内存的2%

### 验证

验证要验证Hypercall的内存访问，运行时指令数和Helper函数的使用

#### 限制运行时指令数

对于全局的Hyperupcall，指令数严格限制在4096个指令之内。

#### 内存访问验证

这一步要验证数据访问在“数据包”之内，“数据包”的范围来源于注册。现有的linux eBPF验证器的能力十分有限。具体方法是探测路径，来检测是否有越界访问。作者弃用了这个方法，选用修改编译器，在内存上加掩码的方法限制内存使用。

#### Helper函数的安全性

Helper函数的检测沿用了eBPF验证器，此外，Helper函数所实现的功能，都是客户机通过别的方法也可以实现的功能。

#### eBPF本身的安全性

eBPF确实会受Spectre影响，但是开了JIT可以缓解已知的攻击。

此外，这个问题是让hyperupcall有优势的点，因为对于Upcalls和Hypercalls来说，Spectre的缓解机制的性能影响非常大。

### 执行

#### Hyperupcall patching

为了减小检测Hyperupcall存在与否带来性能损耗，上了static-keys机制

#### 访问远程CPU

远程CPU写有安全风险

远程CPU读有性能问题

对于性能问题，作者使用同步点机制解决

#### 使用Guest OS锁

## 实现与评估

评估任务

- *验证后的eBPF代码*和*native代码*比，开销如何
- hyperupcall和其它半虚拟化方案的比较
- hyperupcall怎样提高性能、和提供安全、调试服务

![table5](/images/2019-06-06/hu-table5.png)

运行环境

Dell PowerEdge R630 server with Intel E5-2670 CPUs, a Seagate ST1200 disk

实现原型

- Linux v4.8 + KVM
- LLVM 4
- 打开Linux eBPF "JIT"引擎



### Hyperupcall造成的开销

作者修改过的eBPF解释器的性能损耗不会超过250个周期，此外，验证eBPF代码需要的最长时间是67ms

对于TLB的情况，实现的功能是TLB shootdown，由于hyperupcall推出了TLB清空，所以运行的比native code快

(For the TLB use case which handles TLB shootdown to inactive cores, our hyperupcall runs faster than native code since the TLB flush is deferred. )

### TLB Shootdown

![figure2](/images/2019-06-06/hu-figure2.png)

### Discard Free Memory

![figure3](/images/2019-06-06/hu-figure3.png)

上述两种方案，为整个虚拟化环境带了性能提升

### Tracing

#### 实现

Event tracing是一种调试正确性和测试性能开销重要工具。以前的工作有局限，跑在客户机里的trace不能检测Hypervisor事件。有一类收集客户机信息的方法，这类方法不可用于云服务。此外，这些trace收集到的信息也不够多。

为了解决上面的问题，作者在Hypervisor内实现了ftrace，这种ftrace可以监控客户机所有的VM-exit event。

#### 评估

性能上，作者发现这个和原生代码比要多有232个周期的开销，这个开销比要进行上下文切换值得。

通过此trace，作者发现linux中很多没有必要的CPUID，造成了不少无谓的性能开销。



### Kernel Self-Protection

#### 实现

目前的自检方案都有局限性。例如基于nested page table的方案，不可能为guest代码实现受保护内存访问白名单。

作者用Hyperupcall实现了保护机制，每当发生VM exit的时候，就检测是不是有被保护的代码被写。此外还在页面映射Event上加了Hyperupcall，以检查客户页帧。

#### 评估

这种检测给每个VM exit带来了43个周期的性能损耗，这种性能的节约来自于现代CPU避免了频繁的上下文切换。



