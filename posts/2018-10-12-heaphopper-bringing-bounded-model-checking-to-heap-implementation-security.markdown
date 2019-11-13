---
layout: post
title: "HEAPHOPPER: Bringing Bounded Model Checking to Heap Implementation Security"
date: 2018-10-12 14:44:09 +0800
comments: true
categories: 
---

原文：https://seclab.cs.ucsb.edu/media/uploads/papers/sec2018-heap-hopper.pdf

作者：Moritz Eckert , Antonio Bianchi, Ruoyu Wang, Yan Shoshitaishvili , Christopher Kruegel , and Giovanni Vigna 

单位：University of California, Santa Barbara, The University of Iowa, Arizona State University 



作者通过使用符号执行技术，自动化分析堆分配器，根据配置文件定义的情况可以自动生成不同exploitation primitive 的POC

![overview](/images/2018-10-12/overview.png)

### 背景

当程序逻辑存在漏洞时可以攻击堆分配器拿到程序控制权。为了缓解这些攻击，各大堆分配器都在内部进行了安全检查，然而并没有一个标准能评估加入一个补丁是否能缓解攻击。

<!--more-->

以glibc的补丁为例

```c
 /* Take a chunk off a bin list */
 #define unlink(AV, P, BK, FD) {                                            \
+    if (__builtin_expect (chunksize(P) != prev_size (next_chunk(P)), 0))      \
+      malloc_printerr (check_action, "corrupted size vs. prev_size", P, AV);  \
     FD = P->fd;                                                                      \
     BK = P->bk;                                                                      \
     if (__builtin_expect (FD->bk != P || BK->fd != P, 0))  
```

补丁作者加入了一个对size的检查，然而这可以被轻松的绕过，具体利用见

[poison_null_byte.c](https://github.com/shellphish/how2heap/blob/master/glibc_2.26/poison_null_byte.c)

因此作者通过符号执行技术对堆的交互进行了建模，从而可以自动化的评估堆分配器在特定情况下达到exploitation primitive的条件。

### 生成堆交互模型

#### 堆事务

作者将会修改堆的状态的操作定义为事务，对堆的交互分为两种：直接交互和间接交互

直接交互指分配器功能，如malloc，free

间接交互指修改分配的内存区域，比如overflow

##### malloc(M)

HEAPHOPPER 通过传递一个符号化变量对malloc申请的大小进行建模。

一个不可约束的值可能会导致路径爆炸和约束复杂，因此将大小设置为了一个具体的范围，因此符号执行单元将会使用symbolic-but-constrained 变量作为传递给malloc的参数。

通常根据堆分配器不同的执行路径来确定大小，作者开发了一个工具根据libc的执行路径来确定大小选择（然而并没有看见这个工具，源码里面也有提及）。

##### free(F)

如果之前执行过多个malloc事务，HEAPHOPPER将会生成一个不同的序列将其中的每一个作为free事务的参数

##### overflow(O)

在模型中，一个overflow代表一个和堆的间接的交互。作者通过在malloc出的chunk的末尾插入符号内存来实现overflow。和free事务类似，HEAPHOPPER将会为先前分配的chunk创建一个不同的序列作为overflow的目标。和malloc同样的原因，需要限制溢出的长度。在这将会把overflow的大小作为symbolic-but-constrained 处理。HEAPHOPPER 还支持根据不同的场景将overflow的字节设定为特定的范围。

##### use-after-free (UAF) 

我们通过将符号内存写入已经free的chunk作为UAF事务。和之前的事务相似，HEAPHOPPER需要为之前free过的chunk创建一个不同的序列，将有限的字节的数据写到内存中。

##### double-free (DF) 

double-free被建模为对之前释放过的内存执行free

##### fake-free (FF) 

fake-free是释放一个假的，攻击者自定义的chunk。它被建模为free一个完全符号化的内存区域。这个符号化区域的大小必须有个限制。如果可能的话，符号执行单元将会自动确定符号区域的值能通过堆分配器的检查。

#### 堆交互模型

HEAPHOPPER 将结合之前描述的堆事务生成一个交互列表，每个交互对应着堆模型的一条路径。HEAPHOPPER 通过创建所有可能的事务序列的排列来生成交互列表。

在这一步主要关注如何减少生成的事务序列而不丢失会导致产生exploitation primitives 的事务序列。因此，我们假设至少一次对堆进行了误操作（直接或间接），并且假设对堆的良性使用不会产生任何问题。此外，还排除了只有一个间接交互作为事务序列结束的情况，因为间接交互无法修改堆本身，修改堆本身至少需要一个直接交互（？？？？？）。两个会对同一区域放置符号内存的的事务之间必须有影响这片内存的事务。

### 模型检测

#### 识别安全违规行为

##### Overlapping Allocation (OA) 

HEAPHOPPER 使用 SMT求解程序去判断如下条件是否为真

∃B : ((A ≤ B)∧(A+sizeof(A) > B))∨((A ≥ B)∧(B+sizeof(B) > A)) 

A为申请的内存，B为已申请的内存

##### Non-Heap Allocation (NHA) 

提前记录堆分配器使用brk和mmap返回的地址，从而确定堆的范围

##### Arbitrary Write (AW and AWC)

在调用malloc，free时通过检测符号内存区域是否有写入来判断是否有AW或AWC。具体来说,我们查询约束求解器检查是否可以指定的写入特定的内存区域。

### 局限性

#### 模型限制

当前情况下需要手动指定攻击者可以执行的事务。HEAPHOPPER无法对可能发生但是HEAPHOPPER中没有实现的攻击场景进行推理。之前为了减少符号执行样本的数量加入了一些限制条件，这可能会导致HEAPHOPPER错过一些利用。其次一些攻击需要执行很多事务才能把堆设置为理想的状态
