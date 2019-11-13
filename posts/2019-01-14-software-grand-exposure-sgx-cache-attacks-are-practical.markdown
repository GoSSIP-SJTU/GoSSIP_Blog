---
layout: post
title: "Software Grand Exposure: SGX Cache Attacks Are Practical"
date: 2019-01-14 16:10:59 +0800
comments: true
categories: 
---

作者：Ferdinand Brasser, Ahmad-Reza Sadeghi, Urs Muller, Alexandra Dmitrienko, Kari Kostiainen, Srdjan Capkun

单位：System Security Lab, Technische Universitat Darmstadt,  Institute of Information Security, ETH Zurich

原文：https://www.usenix.org/system/files/conference/woot17/woot17-paper-brasser.pdf

出处：USENIX woot17

<hr/>

本文发表在WOOT 2017国际学术会议上，作者介绍了一种Intel SGX可信计算方案的Cache侧信道攻击方法。作者认为这种方法相较现有方法，易于实现且能抵御现有的防御方法。作者着重关注了敏感信息的处理过程，认为这类程序天生难以避免此类攻击，且目前的防御方法均不实用。

<!--more-->

## SGX特性

- 进出内存加密，在Cache和寄存器内部明文
- OS所有行为被记录 可供验证
- AEX（异步安全退出）被中断时擦除寄存器
- 可以禁用PMC（性能计数器）防止侧信道

## 假设

- 可以重放输入重复执行
- 攻击者能控制整个系统，地址空间随机化无效，知道缓存行和内存块的对应关系
- 不能读取enclave的数据，但是知道enclave的代码和初态

## Prime + Probe

![攻击原理示意图](/images/2019-01-14/assets/1544237617178.png)

- 程序特点：根据当前key bit访问不同内存
- Cache信道的共同问题：噪声
- 降低噪声的方法
  - 修改调度器，Victim和Attacker在一个核心上SMT（超线程）运行，防止其他进程的干扰
  - L1 Cache指令和数据分离，自污染相对少。自污染不能彻底避免
  - 无中断执行
    - 修改中断控制器使此核心不响应中断
    - 降低本核心timer频率
    - 需高频并行采样
  - 用PMC直接监视Cache状况，不去读计时器；监视自己的PMC，不受SGX关闭PMC的影响
  - 多轮执行，每轮只监视一个缓存行，从而降低负载，实现高频采样

## 对RSA的攻击实例

- SGX SDK中的`fixed-windows`RSA实现
- 监视一个预计算的乘数表
- 300次重复解密，2048位密钥可以获取70%，足以恢复出剩余部分
- `scatter-and-gather` 方法防御 每次查表都把所有缓存行都过一遍

## 对Primex的攻击实例

处理的对象是一个AGCT序列，敏感的数据为STR，在指定位置上特定短序列的重复次数，如`TAGATAGATAGA`为`TAGA`重复三次。这个特征可用于个体DNA识别。

![Primex预处理过程示意图](/images/2019-01-14/assets/1544238995581.png)

Primex 算法的预处理过程中，会通过一个大小为k的滑动窗口，将序列分为若干的`k-mer`，对于每一个`k-mer`，将其出现位置放在散列表中。对这个过程可以进行攻击。

我们可以监视缓存的使用，推导出每次引用了散列表的哪个元素，从而反推出每个`k-mer`。一个问题是多个散列元素会对应到同一个缓存行，但是考虑到`k-mer`是前后咬合的，通过缓存行访问序列我们仍然可以还原出碱基序列。一个例子是途中，如果先访问 Line 0 再访问 Line 3，那么第一个`k-mer`一定是AT。

![Primex攻击结果分析](/images/2019-01-14/assets/1544239342167.png)

在攻击样例中，作者关心CSF1PO位置上TAGA的重复次数，监视了四个`k-mer`所对应缓存行，将结果对其作图。实线框内四个缓存行均被密集替换，说明此区段出现了TAGA的重复。作者认为该方法能较好防止假阳性。该方法不能精确测量重复次数，作者自己构造了重复序列并处理收集图样进行对比才得出长度。作者认为这个方法的精度能达到±1，此精度条件可以在一千万人口中定位出703人。

## 现有防御

- 禁用Cache：性能损害大，对于科学计算是不可接受的
- 硬件设计：对Cache访问也做随机化，或者把Enclave的Cache独立出来。现有硬件上无解。
- 做混淆，但是Cache空间太小，开销过大
- 定时冲洗Cache，对超线程无效
- 实现加固，对密码算法可行，对科学计算不现实
- 编译器自动加固，不够可靠不够全面
- SGX Shield：做了代码随机化，没做数据随机化，访存还是有特征
  - 本来的目的是防止bug被利用，能提高侧信道利用难度，但不是对侧信道的专门方法
- 攻击检测并终止执行：除了transactional memory（TSX），其他的攻击检测都可以被系统禁用
  - T-SGX：检测是有效的，但是被重放就没用了。有抗重放措施，但本文作者认为可能失效
  - Deja Vu：计时检测中断延迟，计时依赖一个Timer Thread，Timer用TSX保护，问题是Timer自己可能变慢

## 现有SGX攻击

- Page Fault方法
- 已经有Cache方法，但是
  - 一种需要高频中断，容易被检出，还引入了噪声
  - 有一种对L3的方法，权限要求较低，噪声应该会比较多，没有展开讨论
  - 还有一种方法要求Attacker和Victim是同一进程的两个线程，条件较多

## 创新点

- 易于部署
- 免疫现有防御方法
- 关注了对敏感信息的处理过程，这样的程序更难防御，开发者安全水平也更低
