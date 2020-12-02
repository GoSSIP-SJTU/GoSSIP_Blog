---
layout: post
title: "Iodine: Fast Dynamic Taint Tracking Using Rollback-free Optimistic Hybrid Analysis"
date: 2019-04-03 13:03:50 +0800
comments: true
categories: 
---

作者：Subarno Banerjee, David Devecsery † , Peter M. Chen and Satish Narayanasamy

单位：University of Michigan | Georgia Institute of Technology

出处：IEEE S&P 2019

资料：[Paper](http://web.eecs.umich.edu/~pmchen/Rio/papers/banerjee19.pdf)

<hr/>

## 1. Abstract & Introduction

DIFT技术也称为动态污点分析 (Dynamic Taint Analysis) ，可以作为一个安全方案实施，但是实际并没有投入使用，原因就是过高的性能开销。DIFT效率低的原因是需要监控每一个指令，从而监控污点的传播。

本文中，作者提出了一个新的针对DIFT (Dynamic Information-flow Tracking) 的优化方法，叫做Iodine。这个新的技术基于OHA (Optimistic Hybrid Analysis)来减小DIFT的开销，同时不需要回滚（开销就小了）。

本文中作者主要和三种DIFT技术进行了对比：
1. 最原始的DIFT
2. 结合了保守静态分析的DIFT
3. 基于OHA的DIFT（指的就是结合了预测静态分析的DIFT）

作者的关键思路是限制预测静态分析 (Predicted Static Analysis) ，只删掉那些被证明是可以安全删除的monitor (即Safe Elision，作者后面还说明了Noop Elision即为Safe Elision)，从而减少运行时的monitor，在保证性能的同时又保证稳定性，具体的预测静态分析采用的是预测前向污点分析以及传统的后向污点污点。这种思路下，程序事先运行时即使违背了预先假设的Invariant（预先的一些认为会成立的假设），也能正常执行。

<!--more-->

本文中作者的贡献如下：
1. 展示了新的OHA技术来优化DIFT。
2. 解决了旧的基于OHA的DIFT中Invariant被违背时回滚所带来的开销问题。
基于预测的后向静态分析无法保证删掉的monitor都是安全的， 因此后向静态分析用的都是保守的版本。
3. 展示了一种新的profiling技术用于发现Invariant。
4. 新的技术将DIFT的开销减小到了9%，老式的混合分析（即前面提到的第二种）的开销是现在的4.4倍，最原始的DIFT是现在的68倍。

## 2. Design

如下图中展示了四种方案的区别。

![-w887](/images/2019-04-03/media/15536705518788/15537022420910.jpg)


### 2.1 Optimistic Hybrid Taint Analysis

OHA的思想就是静态分析需要把那些实际动态执行时不会执行到的也优化掉。例如图1(c)中，假设所有执行的结果，变量p肯定都是非负数，那么R代码块就肯定不会被执行（这就是一种Invariant）。这种情况下，第2、4、5行的monitor都可以直接删掉，只需要在R里面加上一个Invariant check就可以了。

Iodine的整个Workflow如下图所示。

1. 最开始是一个profiler去收集一系列的候选Invariants，这些Invariants通常是一些动态执行中可能的行为，例如不会执行到的代码等。这些Invariants通常都是几乎正确，但是很难通过静态分析去验证。
2. 然后这些Invariant可以被用来删除没必要的monitor。
3. 还要往程序里插桩，插入Invariant check的monitor
![-w444](/images/2019-04-03/media/15536705518788/15537022226364.jpg)

在大部分的动态执行中，候选Invariant都不会被违背，很稳定。但是一旦这些Invariant被违背了，就需要另外的技术来进行复原了。

### 2.2 Rollback Recover in OHA

例如图1中，假如本来被认为是不会执行到的代码块R被执行了，就需要把前面的第2行代码认为是不稳定的，因为y来源于s。老式的OHA方法在这种情况下，就会完全重新开始动态分析，由于Invariant不会经常被违背，这种回滚在离线分析情景下是可以接受的，例如调试和取证分析。但是对于在线安全分析来说这是不可以接受的。

Iodine通过前向恢复 (Forward Recovery) 来解决这个问题，完全消除了回滚的需要。

### 2.3 Safe Elisions

之所以需要回滚的原因是，被删掉的monitor与潜在的可能会被违背的Invariant之间的依赖关系。作者的想法是找出Safe elision，这些是不存在依赖关系的。Iodine就是只将证明是Safe elision的monitor找出来删掉。

### 2.4 Noop Monitor

Noop Monitor即为Safe Elision，Noop Monitor指的就是不会修改Analysis Metadata State的monitor。

### 2.5 Elisions in Predicated Forward Analysis are Safe

预测前向分析得到的Elision都是Safe Elision，这个可以从图1-c中的例子里直观的感受到。通过预测前向分析，会扫描每条指令，如果源操作不会被污染，那就认为是Elision。所以例子中预测前向分析只会认为1，5，6为Elission

### 2.6 Elisions in Predicated Backward Analysis may not be Safe

预测后向分析得到的Elision不一定是Safe Elision。从图1-c的例子中可以看出来，预测后向分析会把2也认为是Elision。

### 2.7 Rollback-Free Optimistic Hybrid Taint Analysis

Iodine使用了预测前向污点分析以及传统的后向污点污点。inv_check这里会插桩，插入一个分支，如果真的发现了违背Invariant的情况，就会从这个safe-path（lodine分析得到的path）跳转到slow-path（老式的Hybrid analysis）。

### 3. Implementation

作者的Iodine就是rollback-free OHA的一个实现。Profiler和Dynamic Analysis Instrumenter用LLVM3.9实现。目前Iodine只支持C语言。最终的插桩用的是LLVM的Data Flow Sanitizer实现。

#### 3.1 Specifying Information-Flow Policies

作者设计了一个可配置的污点策略，这个策略里把所有的外部输入都标记为Taint Source。也允许将标准输出作为sink，例如Terminal，File，Socket。另外策略是可配置的，也就是可以手动修改策略。例如要标记一个变量password为Taint Source，就可以用taint(password) = secret，这样就把secret这个Taint Mark附到password变量上了。

### 3.2 Static Taint and Pointer Analysis

静态污点分析要在给定了Taint Source，Taint Sink，Propagation Policy的情况下计算污点的传播。作者用的是已有的Whole-program Context-sensitive Flow-sensitive Data-ﬂow May-analysis。通过构建DUG（deﬁnition-use graph），每一个节点代表了一个定义了输出数据的指令，边用来连接使用了这个定义的节点。

一旦DUG建立好了，分析就遵循Forward optimizations和Backward optimizations两种优化方式。

### 3.3 Optimistic Hybrid Taint Analysis

用LLVM DFSan来插入monitor，DFSan是一个动态的DIFT工具。

### 3.4 Forward Recovery Mechanism

每一个函数都实现了一份Fast-path代码和Slow-path代码，如下图所示

![](/images/2019-04-03/media/15536705518788/15537138359314.jpg)

## 4. Evaluation

### 4.1 实验设置

作者用几个安全敏感的Real world应用程序对Iodine进行测试。Benchmark suite由以下组成：
1. Postfix mail server test generators
2. nginx, thttpd
3. redis
4. vim
5. gzip

作者与一个老式的Hybrid IFT工具（Information Flow Tracking），以及一个naive的动态IFT工具进行对比。最后的结果如下图所示。
### 4.2 IFT Security Policies

作者用真实世界的Taint policies进行实验，来证明Iodine的效率。

可以看到Iodine在性能上的提升很高。

![](/images/2019-04-03/media/15536705518788/15537147313401.jpg)

### 4.3 Generic Information-Flow Policies

由于Realisitic Taint Policies是有限的，作者采取了两个策略来增加Taint Policy：
1. Some-to-some：随机选取Taint source的一个子集作为Taint source，Taint sink就为原来的Taint sink
2. Some-to-all：随机选取Taint source的一个子集作为Taint source，Taint sink设置为所有的指令

测试的结果和前面的差不多，能显著的减少运行时的时间开销，从下图中能看出来

![-w660](/images/2019-04-03/media/15536705518788/15537420057392.jpg)


### 4.3 Memory Overheads

Iodine会导致程序的代码有两个版本，一个Fast-path，一个Slow-path，但是在同一时间内实际上只有一个版本的代码会被执行，因此对caching和性能的影响不大。


