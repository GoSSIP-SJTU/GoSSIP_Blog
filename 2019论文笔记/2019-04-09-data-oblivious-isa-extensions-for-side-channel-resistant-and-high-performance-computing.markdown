---
layout: post
title: "Data Oblivious ISA Extensions for Side Channel-Resistant and High Performance Computing"
date: 2019-04-09 17:03:13 +0800
comments: true
categories: 
---

作者：Jiyong Yu, Lucas Hsiung, Mohamad El Hajj, Christopher W. Fletcher

单位：University of Illinois at Urbana-Champaign

会议：NDSS 2019

原文链接：https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_05B-4_Yu_paper.pdf

## 摘要

- 指出了现有Data Oblivious Code（秘密信息不会留下trace的程序，健忘的）的安全性问题和性能问题
- 在BOOM（Berkeley Out-of-Order Machine）基础上设计了一个可乱序执行可预测执行（允许常见优化）的OISA
- 可以证明达到了BitCycle级的路径不可分辨性
- 进行了性能评估，证明其性能优势

<!--more-->
## 定义

### Threat Model

- 假定有enclave保护（不能直接读数据）
- 可以防御特权攻击者
- 不考虑硬件侧信道和计算完整性问题

### 安全性定义：

- 定义3.1：秘密输入的隐私性

  同一程序，使用相同的公开输入，在相同架构上执行，用同样的方法观测执行路径，要求对于不同的秘密输入，观测所得路径应当计算上不可区分。

- 定义3.2：BitCycle可观测性：观测的精度可以达到逐cycle、逐bit

  Threat Model要求不能直接读每个bit的数据，但是可以知道每个cycle时每个bit是否由秘密信息决定，数据或者地址受影响都算。

- 本文目标是使用3.2的观测方法达成3.1定义的安全性

## OISA的必要性（现有Data Oblivious Code存在的问题）

![双倍的内存访问](/images/2019-04-09//assets\1551857337902.png)

### 性能问题：

- 时间平衡
- 更多的内存访问，防止cache trace（见上图）
- 用基础指令重新实现变时长硬件指令

![提前脱敏示例](/images/2019-04-09/assets\1551857390199.png)

### 安全问题：

- 分支预测导致提前脱敏，进入脱敏过程的数据失去保护，可能通过寄存器重命名泄露（见上图）
- 亚地址优化：有的常数时间算法依赖于操作同一缓存行中的偏移地址，这个做法在有的架构上不可靠
- 变时长算术指令：计算时长可能和参数有关
- 微架构实现的不确定性，如`cmov`可能被实现为分支加move
- 其他面向数据的优化，只要不区分数据是否是秘密，就可能引起问题

## 设计与实现

- 每个指令都描述了每个操作数能否安全接受秘密信息（指令级的安全性保障）
- 尽可能进行优化，除非当前数据不允许

### 规则设计

- 4.1：数据都必须带标签（公开还是机密），标签必须与数据一起就位
- 4.2：秘密信息作为安全操作数：执行，并掩盖各种副作用，包括拖时间、吞异常（脱敏后才释放异常）
- 4.3：秘密信息作为不安全操作数：拒绝执行
- 4.4：公开数据作为操作数：正常执行，可启用（仅面向公开操作数的）优化
- 标签的传递：若产生的结果完全由公开操作数决定，则结果属性为public
- 数据脱敏：必须串行执行（防止提前脱敏）

![OISA扩展指令表](/images/2019-04-09/assets\1551859084677.png)

### 指令类别：

- 接受安全操作数的算术指令
- 随机数生成指令
- 安全的`cmov`指令
- 不安全的load/store
- 安全的load/store（memory oblivious指令）
- 敏感数据声明/脱敏指令
- 其他原有指令保留，但只能接受不安全操作数（所以跳转都是不安全的）

### Memory Oblivious Extension：

- 架构应该提供oblivious memory partition（OMP），只有安全的load/store能操作
- 三种操作模式：
  - OMP：相关数据结构整个分配在OMP上
  - ORAM：使用ZeroTrace库实现ORAM，ZeroTrace的一些数据结构存放在OMP上从而得以提速
  - SCAN：fallback，每个元素读一遍

![BOOM结构示意图](/images/2019-04-09/assets\1551860369579.png)

![Label Station示意图](/images/2019-04-09/assets\1551860420483.png)

### 实现

- 内存中label和数据分开存，送入CPU时组合
- 每个执行单元加了一个前置的label station来进行权限检查
- OMP从cache上动态分配，分配了就会阻止其他代码使用这段cache

## 证明

简单来说，需要证明

- 每个指令的时长和副作用等与秘密信息无关（已经被label check实现了）
- 执行的指令流和秘密信息无关（秘密信息无法污染PC）

首先证明AOOM（Abstract-）满足WordStage法（文中定义6.1）下的安全性：

- Execution traces定义为一个四元组序列：
  - stage：Fetch，Execute，Retire三阶段之一
  - pc：留空表示数据声明
  - squash：stage=Execute时，若出现label错误或预测错误，设为true，否则为false
  - update：Write(addr, data, label)，若无需写入addr为空
- 模拟流水线和多处理单元：同一时间可以有多个traces，stage & pc不同
- 模拟乱序执行和预测执行：定义SCHEDULE和PREDICT函数
  - SCHEDULE返回下一步的行为，若返回tracestep编号则执行该tracestep的下一阶段，若返回空则调用PREDICT返回下一条要加载的语句
  - SCHEDULE和PREDICT的参数都是移除了update字段的traces
  - 不同的SCHEDULE和PREDICT实现可以定义出不同特征的处理器，在此只限制Fetch和Retire必须有序，两个函数都是确定性的
- 模拟储存器状态：储存器状态完全由traces定义
- 定义WordStage观测：对public数据观测值，对秘密数据观测label即可
- 归纳法加分类讨论，先证明初态相同，然后证明执行过程中对每个分支的选择都会相同

接下来需要把AOOM推广到BOOM，把WordStage推广到BitCycle。这一部分的严格证明属于future work。

- BOOM的SCHEDULE和PREDICT的参数是AOOM的子集
- 执行单元的资源限制可以通过向SCHEDULE施加约束来模拟
- 增加更多的stage和inst type，并没有实质性影响
- 支持不同的指令延时，可以通过增加Execute的阶段数来模拟
- 可以通过一个以历史访问内存地址列表为输入的新函数来模拟cache的存在
- WordStage => WordCycle => BitCycle

## 评估

### 面积评估

![面积评估](/images/2019-04-09/assets\1552030193878.png)

新增面积主要来源：

- 动态数据流跟踪DIFT
- Label Station
- OMP划分逻辑
- 随机数生成：目前使用AES迭代，仅此一项就新增了3%，换用TRNG有望优化

### 性能评估

使用软件模拟，比较了三种平台：

- 原始ISA，运行非oblivious代码；用作现有应用的基准
- OISA，不启用OMP，运行oblivious代码；用作现有oblivious代码的基准（只有安全上的改进，性能改进主要还是依托OMP）
- 启用OMP的OISA

选取的测试程序主要分三类，每个程序都分别进行了小数据规模和大数据规模的测试：

- 本来就oblivious的程序
- 依赖oblivious sort的程序
- 依赖oblivious memory的程序

![测试规模表](/images/2019-04-09/assets\1552030876421.png)

测试的结果显示，只要数据能完全在OMP中处理，OISA的加速效果是非常好的。

![测试结果](/images/2019-04-09/assets\1552030975793.png)

作者提出两个改进的方向：

- 增大OMP的大小，测试中只在L1上分配了32K的OMP，多级的、更大的OMP会有好处
- 将常见kernel指令化（如sort）

### Case study: constant time AES

- 基准：不安全平台上的查表AES
- 使用OISA&OMP的查表AES：2.17×
- 安全的bitslice AES：9.6×

作者认为2.17×的减慢还包括编译器不能很好的优化新指令前后语句的因素，所以OISA很有前景

### Case Study: ZeroTrace

![ZeroTrace分析结果](/images/2019-04-09/assets\1552031897171.png)

ZeroTrace是一个现有的Oblivious Memory操作库。OISA会使用OMP、SCAN、ORAM三种操作模式，选择取决于数据量大小。ORAM即使用ZeroTrace库，但是会将其stash存放在OMP中，stash大小不会随着数据量增长而增长。由上图可见，OISA能够利用各种策略的优势，且将stash存放在OMP中对于提高ZeroTrace的效率有一定的作用。