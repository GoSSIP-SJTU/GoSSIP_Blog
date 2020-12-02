---
layout: post
title: "When Function Signature Recovery Meets Compiler Optimization"
date: 2020-10-23 15:34:34 +0800
comments: true
categories: 
---

> 作者：Yan Lin, Debin Gao
>
> 单位：Singapore Management University
>
> 会议：IEEE Symposium on Security and Privacy 2021
>
> 原文：https://www.computer.org/csdl/proceedings-article/sp/2021/893400a091/1mbmHtHLPAA

# ABSTRACT

总所周知，为了提升程序运行时的效率与性能，现代编译器会对程序进行各种各样的优化。但凡事必有利弊，在带来性能提升的同时，编译器优化也可能会产生一些不好的影响。本文主要讲述的就是编译器优化对反编译过程中函数签名（function signature) 识别产生的（正面与负面）的影响。在程序分析中，识别间接调用是实现控制流完整性 （CFI) 中十分重要的一环。对于二进制程序的 CFI，其中一种识别间接调用方式就是分析函数的调用规约（call convention) 并生成函数签名 （function signature)， 根据函数签名将 caller 与 callee 进行配对，而编译器的优化会对这一方式带来一些影响。为了使得这一问题得到重视，作者系统地分析了编译器优化对于函数签名带来的改变，并相应提出了相应的改进策略。

<!-- more -->

# INTRODUCTION

现今利用函数签名识别实现二进制程序 CFI 工具有 **TypeArmor** 与 **τCFI** ,所以本篇文章以上述两款工具作为分析对象。

总的来说，本篇文章作者主要的贡献如下:

* 总结了编译器优化对函数签名识别的影响，并在1334个现实软件中进行了分析与测试
* 针对分析的结果，提出了一些能够提高识别准确率的改进策略


# FUNCTION SIGNATURE RECOVERY

在特定平台中，函数调用有其指定的传参规约。比如说在 Linux x86-64 中，参数的按顺序传入`rdi, rsi, rdx, rcx, r8, r9` 寄存器中，同时 `XMM0 - XMM7` 传递浮点数参数，其余的参数会被按顺序压入栈中。而返回值则会被存放到 rax 寄存器中。根据传参规约，反编译器可以根据在调用点以及被调用点的参数寄存器的使用情况，可以对每个函数提取出一个如下的状态列表，进而分析得到函数签名。



## Callee

在被调用点（函数开头），反编译器会识别第一条对参数寄存器进行访问的指令，并提取出形如下图的状态列表。
![](/images/2020-10-23/uploads/upload_0be0d8661b6ca3dff234eca9fcc9e473.png)



从该签名中可以得知，前5个参数寄存器的值都被存入到了其它非栈内存或者寄存器中，第六个参数的值则被放入到了栈上。

接下来，对于提取到的状态列表，根据相应的规则来获取函数签名。

识别函数签名有两个过程
1. 区分该函数是否是**变参函数**
2. 识别参数的个数

在 x86-64 的代码中，变参函数有一个特征：会将所有的参数都存入到栈中。

![](/images/2020-10-23/uploads/upload_b86fdefff82b303f523ce15a4c57200b.png)

举个例子，上图是 gcc 编译的一个有3个固定参数的变参函数的汇编代码。可以看到该函数将所有的参数寄存器中的内容都存入到了栈中。

根据这一规律，**TypeArmor** 与 **τCFI** 工具简单地将判断 **r9d（第六个参数寄存器）** 是否被压入栈中。如果是，则认为该函数是一个变参函数。

接来下需要识别函数参数的个数。
+ 普通函数：以最后一个被读取的参数寄存器作为参数个数（比如 rdx 被读取，且 rcx 未被读取，则认为参数个数是3）。
+ 变参函数：固定参数会在栈地址中连续排布，可以根据这一个特征识别固定参数的个数。

![](/images/2020-10-23/uploads/upload_4398182f30be14ea8b42ae8d2f644071.png)


## caller

对于 caller 的函数签名的识别方式与 callee 处相似。不同之处**在于 callee 处观测的是寄存器的值是否被读取，而 caller 处则需要观测寄存器的值是否被写入**。


## caller 与 callee 的匹配规则

在得到了 callee 与 caller 的函数签名之后，下一步需要对每个 caller ，匹配出可能的 callee 对象。匹配与否取决于参数的个数以及参数的位数。
+ 参数的个数相同
+ caller 处的参数的长度要大于等于 callee 处参数的长度


# INFLUENCE OF COMPILER OPTIMIZATION

编译器优化对于 callee 处的函数签名以及 caller 处的函数签名都有着不同的影响

## Complications at Callees

### A: Misidentifying variadic functions

1. 第一种情况指的是**将一个普通函数识别成了变参函数**。因为一个函数是否是变参函数，取决于 r9d 寄存器的值是否被存放入栈中。如果一个普通函数的参数长度大于6，那么就有可能会将一个普通函数误判成为一个变参函数。
    ![](/images/2020-10-23/uploads/upload_7a6e5e5299f528d7fdee1f3e9361e0ed.png)
      如上所示，该函数是一个7个参数的普通函数，但是因为 r9 寄存器的值被存放到栈中，因此该函数被误判成了一个变参函数。


2. 另一个误判的情况是，假如变参函数的参数寄存器因为编译器优化的原因，没有被存放入栈中，则会**将一个变参函数误判成为普通函数**。
    ![](/images/2020-10-23/uploads/upload_d9a0fd34fa8087d7e0590d50043e3efa.png)
    如上图所示，因为 r9 寄存器的值没有被存放入栈中，该函数被判断成为一个普通函数。而实际上该函数是接收一个参数的变参函数。

**以上两个问题的根源在于：以 r9 寄存器是否被压栈作为区分变参函数的唯一标准是不够的。**

#### 解决方案：

作者的改进基于如下的 insight
+ 可变参数通过 va_arg 获取，**va_arg** 会通过一个指向栈地址的指针访问参数
+ 定长参数通过 [rbp+offset] 获取

举个例子

```c++
int test(long int a, int c, char *fmt, ...)
{
    va_list ap; 
    va_start(ap, fmt);
    int b;
    while(1)
        b=va_arg(ap, int);
}
```
前三个参数都是固定参数，获取可变参数需要调用第七行 **va_arg(ap, int)**
编译之后的汇编代码如下
![](/images/2020-10-23/uploads/upload_835e295f38b0b89f1ea9c3988fad7ce1.png)
可见见到，通过对 rax 寄存器解引用获取变量b的值。

根据这一差异，可以更为准确判断一个函数是否是变参函数
+ 如果通过解引用方式获取参数，该函数是一个变参函数


笔者想法：IDA反编译的信息已经足够区分变参函数和普通函数。

```c++
 gcc_va_list va; // [rsp+28h] [rbp-D0h]

  va_start(va, a6);
  va[0].gp_offset = 24;
  while ( 1 )
  {
    if ( va[0].gp_offset > 0x2F )
      va[0].overflow_arg_area = (char *)va[0].overflow_arg_area + 8;
    else
      va[0].gp_offset += 8;
  }
```
可以见到，ida 不仅能够准确识别 va_start，而且还可以通过 **va[0].gp_offset = 24;** 得知定长参数的个数为3（24/8）。


---

### B: Missing argument-reading instructions
如果某个参数没有被使用，在 dead-code elimination 优化后，那么该参数将不会被传入到栈中，导致识别出来的参数个数与实际值不匹配。

![](/images/2020-10-23/uploads/upload_22b4a756f9fa1754d86707a449ad8e18.png)

比如上面这个例子，第一个和第三个没有被使用，所以相应的汇编代码没有对 rdi 与 rdx 寄存器进行操作，因此识别出来的参数的个数与实际值也就不相同。

对于这个问题，作者没有提出解决方案。

---

### C: Misidentifying rdx as an argument

rdx 寄存器除了用于传参，还可以被用于除法运算指令当中。如果程序中出现了相应的指令，则可能会将该寄存器的值误判为一个参数。
![](/images/2020-10-23/uploads/upload_edbab359a7cf84ec8c21c4ab80f9f1e9.png)

#### 解决方案：

在调用一个另一个函数之后再读取 rdx 寄存器，将不再把 rdx 寄存器的值当作是参数。

---

### D:Argument (width) promotion

为了减少指令的长度，编译器可能会采用一些64位指令来对32位数据进行操作。常见的情况可以是用 push 指令或者是 lea 指令存储32位长度的数据。

#### 解决方案：

作者为了应对这一问题，提出了下面的规则。
![](/images/2020-10-23/uploads/upload_856a0a85dd8c8dad9c0f6431f9800f67.png)

---

## Complications at Callers

### A: Missing argument-writing instructions

为了减少不必要的指令，编译器可能会将一些多余的传参指令消除。比如

+ **参数在上一个函数调用被存入寄存器中**。这种情况下，在调用目标函数之前将不会重复为寄存器赋值

![](/images/2020-10-23/uploads/upload_82cfef51fe681e562549b76e991ef2fa.png)

比方说在如上例子中，第三个参数的值已经被除法语句提前存放在了 rdx 寄存器中，因此将不会再重复赋值寄存器。

对于这一问题，作者没有提出解决方案。

---

### B: Registers storing temporary values

有时候参数寄存器仅仅只是被用于存放临时中间变量，可能会被误认为在传参。
![](/images/2020-10-23/uploads/upload_169468262f6f6c53cd33646f2fbe4a66.png)

比如上图 rcx 寄存器用来存储中间值，并不用于传参。但是反编译器无法区分，因此会把本次函数调用认作是传入3个参数。

#### 解决方案：

如果给一个参数寄存器赋值之后，又将该寄存器的值传给其它的寄存器，则不再把对该寄存器的赋值当作是传参。

---

### C: Argument (width) demotion

因为在 x86-64 架构中，对低32位寄存器赋值会自动将高32位清零。所以假如传入的参数虽然是一个64位类型，但是长度比较小，这时候编译器会采用32位的操作来减少指令的长度。

![](/images/2020-10-23/uploads/upload_cfec948fac88c595d0d32ffbfd43070e.png)

比方说上面的两个立即数都用32位寄存器进行传参

#### 解决方案：

![](/images/2020-10-23/uploads/upload_856a0a85dd8c8dad9c0f6431f9800f67.png)

# EVALUATION

## EXPERIMENTAL RESULTS ON REAL-WORLD PROGRAMS

为了验证上述的这些编译器优化对于实际程序的影响，作者选取了552个C程序以及792个C++程序，分别用gcc 和 clang 编译器在 O0-O3 的优化下进行测试。测试选取的对象是 TypeArmor，τCF 以及 Ghidra 反汇编工具。


### Complications at Callees

![](/images/2020-10-23/uploads/upload_a69b5d45557d2010d5e5919d48de40be.png)

文章中总共用四个图来分别表示 clang 与 gcc 编译器对于 c与c++ 程序的影响，上图是用 clang 编译 c 程序时带来的影响。

纵轴表示出现**overestimation 以及 underestimation 的概率**
横轴表示**不同的反编译器与编译优化的结合**



结果表明，无论是在哪种情况下，基本上是编译器的优化级别越高，误判的概率也就越高。

其中，**Missing argument-reading instructions(被调用函数的参数寄存器没有被使用)** 带来的误判是最高的。主要常见于 C++ 程序中，假如 this 指针在类成员函数中没有被使用，编译生成的代码中将不会对相应寄存器进行操作。

其余的像 Nor2Var（普通函数识别成变参函数）, lea/push, rdx（被用于除法）, Var2Nor（变参函数被识别成普通函数） 都会对函数签名的识别产生影响。

### Complications at Callers

![](/images/2020-10-23/uploads/upload_599732d4478da9fc05491e0579d488fd.png)

与 callee 的情况类似，在不同的情况下，编译器优化对函数签名的识别也有着相应的影响。有的影响会使得识别出来的函数参数个数偏高，有的影响会使得识别的参数个数偏低，编译器优化在不同级别的优化下以及对于不同的工具产生影响各异。

## Evaluation of Improved Policies

作者用了三个指标对优化效果进行衡量。

### Overestimation & Underestimation 

作者的实验证明，改进后优化方案完全消除了 **VarOver, rdx, lea** 这三种形式的误判。
![](/images/2020-10-23/uploads/upload_c179a3cce108017d519cea0679190e97.png)
举例来说，从上图可见， VarOver 对于 improved 后的反编译器测试没有误判。

对于其余的情况，比如 **Imm, Null, Push 以及 Nor2Var** 带来的误判也有所降低，其中 **Nor2Var** 的误判概率从 3.3% 降低至 1.2%。

### Indirect Call Targets

![](/images/2020-10-23/uploads/upload_50bb601e5f7ff66821eb055512d205ff.png)

上图表示的是执行 CFI 保护之后，平均每一个 indirect call 的可允许跳转目标的个数。从最后一行可以看出，作者改进之后，允许跳转的目标为 353 个，相比其它基于二进制的 CFI 工具有一定改进。其中 IFCC 与 LLVM-CFI 是源码级别的 CFI 工具，因此效果会更好。

### Gadgets Disable

Main-Loop Gadgets (ML-G) 和  RECursive Gadgets （REC-G） 是在别的论文中提到的两类可以用以攻击 CFI 的 gadget，下图中的数字指的是识别出的 gadget 数量与对应 CFI 策略能够抵御的gadget 数量。
![](/images/2020-10-23/uploads/upload_6b7b5c369a1c1759220153ba334190e0.png)


比如对于 ML-G gadget 而言，平均总共有 83 个可用的 gadget, 作者改进后的 CFI 策略能够抵御其中65个gadget,相比 τCFI 的56个有提升。

