# SMOKE: Scalable Path-Sensitive Memory Leak Detection for Millions of Lines of Code

> 作者：Gang Fan∗, Rongxin Wu∗, Qingkai Shi∗, Xiao Xiao†, Jinguo Zhou†, Charles Zhang∗
>
> 单位：∗Hong Kong University of Science and Technology , †Sourcebrella Inc. 
>
> 会议：International Conference on Software Engineering 2019
>
> 论文：[Scalable Path-Sensitive Memory Leak Detection for Millions of Lines of Code](https://gangfan.github.io/assets/papers/gang_smoke_icse2019_preprint.pdf)

## Abstract

使用值流图（value-flow graph) 进行 memory leak 漏洞检测的方法存在效率低下的问题。为了提升检测的效率，SMOKE 采用自行设计的 use-flow graph  ，将内存泄漏漏洞检测转换成有限状态机上的状态转换问题。实验表明，SMOKE 的运行效率比现有的工具要快上 5.2 到 22.8 倍，且误报率仅为 24.4%。

<!-- more -->

## Value-flow graph

值流图表示的是内存对象在程序中的流向关系，我们可以把它简单理解成对特定内存对象进行操作的程序切片。

![](/images/2020-12-08/upload_dcd6cf4147a5bcfc1330a7ca0c9168a8.png)


下图是对上述代码进行分析之后针对 p 生成的程序值流图。

![](/images/2020-12-08/upload_d9d3870a0837bf9cba4df5199d11c2ef.png)

为了检测内存泄漏漏洞，需要判断是否所有的程序分支都会释放该内存对象。传统依赖值流图进行内存漏洞检测的方式如下：

1. 在值流图上，寻找从 source 到 sink 的所有路径。（上图例子中只有一条路径)
2. 将值流图上寻找到的路径对应到控制流图中，并记录经过的条件分支。（如上图中，S<sub>2</sub>->S<sub>7</sub>在值流图中可以搜索到两条路径，分别是 **$\neg c$<sub>1</sub> $\cap$ c<sub>2</sub>** 和 **$\neg c$<sub>1</sub> $\cap$ $\neg c$<sub>2</sub>**）
3. 计算这些条件分支的并集，判断是否为1。（为1则代表覆盖了所有路径，表示不存在 memory leak）


然而，计算上述逻辑表达式是一件十分耗时的工作，被证明是一个NP困难问题，现有的最优算法的时间复杂度为$O((2 − \frac2 {k+1}$ )<sup>n</sup>)。

## Finite-state machine

![](/images/2020-12-08/upload_8343a66467f187b3d8a44a6f0bd48cef.png)

与计算逻辑表达式的方式不同，SMOKE 采用状态转移的方式检测 memory leak。针对 memory leak，SMOKE 定义了内存对象的4种状态。

+ Allocated : 内存对象刚被创建
+ Freed : 内存对象已被释放
+ Error : 内存对象没被释放就离开了其作用域(memory leak)，或是内存对象被重复释放（double free)
+ Exit : 正常退出

然而，由于值流图（value-flow graph）缺失了流敏感的信息，因此不适于上述的检测方式。


```c
1 #include <stdlib.h>
2 int main()
3 {
4     int *a = malloc(0x30);
5     free(a);
6     free(a);
7 }
```

比如说，上述代码对应的 value-flow graph 如下所示, 可以见到，从图中我们得不到两个free语句的先后关系。
![](/images/2020-12-08/upload_800b17cc5d0583ac7df122e6753af31e.png)

因此 SMOKE 的开发者设计了具备流敏感信息的 use-flow graph，利用其进行上面所述的状态转移的检测。

## Use-flow graph

![](/images/2020-12-08/upload_eef3bb24284aa1848aee7781f38794a1.png)

+ use-flow graph 中保留了如下四种类型的结点
  1. malloc node（s2) / free node (s6, s11) 以及表示内存对象脱离作用域的 out-of-scope node (s9)
  2. 表示实际参数传递的结点（s3)
  3. 表示形式参数的结点 (s10)
  4. 表示路径交汇的结点 （s7,s12）

+ 结点A与B之间存在边，表示A->B在控制流上可达且A与B指向同一个内存对象。

可以见到，在 use-flow graph 中，我们可以获取到语句先后顺序关系的信息。

建图方式：use-flow graph 实际上是简化版的 control-flow graph, SMOKE 根据指针分析的结果，对 CFG 进行重构而生成。

![](/images/2020-12-08/upload_165a48c6dcfe0bb7abc346b2a9a31bc5.png)


## Motivating example

![](/images/2020-12-08/upload_685cd7801cb3900eabc146f2bcc9fbfe.png)
![](/images/2020-12-08/upload_c07fb2dab9216b22465bc40c879acbeb.png)
![](/images/2020-12-08/upload_71699ed3934bb80b6e4ad085cacb0223.png)


## Path-sensitive

通过上面所述的方式找到了一系列的存在漏洞的路径之后，还需要进一步过滤一些由于路径不敏感而带来的不可达路径。

![](/images/2020-12-08/upload_bd0a2d4a108bf4eb85bbdca8595e7354.png)



对于上述程序片段，SMOKE 经过第一步检测，认为程序可能存在 double free 漏洞，但是实际上产生 double free 的路径是一条不可达的路径。

对于这个问题，SMOKE 会使用约束求解器进一步对分支条件信息进行过滤。比如上述路径 C $\cap$ $\neg$C 的计算结果为非，因此这条路径将会被过滤掉。

SMOKE 的约束求解分为两个阶段

+ 在第一阶段，只会简单判断路径上是否存在两个互斥的条件约束。比如上述例子中，C 和 $\neg$C互斥，所以该路径被判断为不可达。
+ 假如路径上的条件约束没有明显的互斥关系，就需要使用像 Z3 一样的 SMT 约束求解器进行求解。



## Evaluation

CPU：Intel Xeon Platinum 8175M * 2 （总共48核）

内存：384G

硬盘：840GB NVMe disk *4

宿主机： Ubuntu 18.04.2 with kernel 4.15.0-1044-aws

测试对象； SMOKE, SABER, PINPOINT，CSA，INFER

测试集： SPEC CINT2000，17个开源软件

### Scalability

![](/images/2020-12-08/upload_cf26b645b9acb932d4c9aeea85db4d23.png)

![](/images/2020-12-08/upload_c7c653253a4956da488098b466f44646.png)

对于17个开源软件（其中8个项目的源代码大于100万行），SMOKE 均在40分钟之内检测完毕。 
平均下来，SMOKE 运行效率比 SABER 快5.2倍，比 PINPOINT 快13倍，比 CSA 快12.4倍，比 INFER 快22.8倍。
此外，其余的工具在检测大型项目时会出现内存不足而停止运行的问题，或者是遇到 segment fault，只有 SMOKE 成功对17个开源项目均检测完毕。 

### Precision

![](/images/2020-12-08/upload_4cd77062d678477e77d3f0ab47e2147c.png)


对于 benchmark 以及开源项目，SMOKE 总共发现了209个 memory leak 的报错，其中有51个误报，总体的误报率为24.4%。
在论及 SMOKE 为何能有如此低的误报率，作者认为这主要得益于在第二步使用约束求解器过滤不可达路径。
SABER之所以有更高的误报率，是因为它没有采用相似的方式过滤不可达路径。
而 CSA 与 PINPOINT 采用了与 SMOKE 相似的方法，所以误报率相比较而言会低一些。

在上述表格所记录的漏洞中，有一个 Bftpd 的内存泄漏漏洞因为其严重危害性被接收至 CVE (CVE-2017-16892)。

作者将其余的漏洞详细信息放至如下的网址中：
https://smokeml.github.io/data/

## Summary

1. SMOKE 主要的亮点在于其构造了一个具备流敏感信息的 use-flow graph，并基于 use-flow graph 采用状态转移的方式检测漏洞，因此大大缩短了检测漏洞的耗时。

2. 其次，采用约束求解器对 infeasible path 进行过滤,对于降低误报率有着比较明显的效果。