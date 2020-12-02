---
layout: post
title: "RetroWrite: Statically Instrumenting COTS Binaries for Fuzzing and Sanitization"
date: 2019-11-05 22:12:15 -0500
comments: true
categories: 
---
作者：Sushant Dinesh, Nathan Burow, Dongyan Xu, Mathias Payer

单位：Purdue University, EPFL

会议：S&P'20

出处：https://nebelwelt.net/publications/files/20Oakland.pdf

------

## Abstract

目前，对于闭源的二进制文件的自动化漏洞挖掘方式为基于动态二进制插桩的模糊测试(fuzzer)和内存安全性分析(sanitizer)，但其有很大的性能开销。目前的静态二进制文件改写技术因为不能完全恢复重定位信息(labels)，对于再编译出的二进制文件并不能保证程序的完整性，可能会导致程序功能异常、漏洞无法正常复现等问题。

本文主要作出了以下贡献：

- 实现了一个可以保证64位PIC程序soundness、completeness的静态二进制改写框架
- 实现了一个用于AFL插桩的pass，使得插桩后的程序能够使用AFL进行模糊测试；并与基于编译器插桩的AFL拥有同样的性能
- 还实现了一个用于ASAN(Address Sanitizer)插桩的pass，使得程序能够使用ASAN进行内存安全性分析，较目前对于二进制文件的内存安全分析(Valgrind’s memcheck)性能提升了三倍

<!--more-->

> 保证control flow, instruction selection, and register and memory access patterns的完整性



> 目前针对binary的自动化测试方案：
>
> - 黑盒 fuzzing
> - 依赖DBT的动态插桩
> - unsound的静态改写技术



> 二进制改写技术：
>
> - recompilation  转成IR再编译
> - trampolines   类似于inline hook
> - reassembleable assembly



## Background

1. 反汇编

   - 线性扫描(linear sweep)，RetroWrite使用linear sweep反汇编
   - 递归下降(recursive descent)

2. 二进制改写技术

   根据改写的方式可以分为两种：

   - 动态二进制翻译(Dynamic Binary Translation, DBT)
     - Intel PIN, DynamoRIO, QEMU, DynInst, libdetox, and Valgrind
   - 静态二进制改写(Static Binary Rewriting)

   DBT运行时开销过大；静态二进制改写依赖于静态分析，目前的方案不够准确且复杂还不具备扩展性。

   

3. 再汇编

   由反汇编器生成的汇编代码，代码和数据的地址都被硬编码了；如果其位置发生变化则会破坏对它的引用。

   而编译器生成的汇编文件则使用label来确定跳转的位置、使用的变量...，一般不会使用硬编码的地址

   再汇编的关键在于，恢复重定位标签，将硬编码的地址转换成使用label的引用，且不能将常量误分类成标签/地址。

   ![image-20191105153853244](/images/2019-11-06/image-20191105153853244.png)

   ![image-20191030162130390](/images/2019-11-06/image-20191030162130390.png)

   <center>由编译器生成的汇编代码</center>

4. 位置无关代码(PIC)

   能够被加载到任意虚拟地址执行的。

   X86_64中的PIC使用rip来进行相对寻址，而不会出现硬编码的地址

![image-20191030161754934](/images/2019-11-06/image-20191030161754934.png)

<center>no PIC</center>

![image-20191030161822200](/images/2019-11-06/image-20191030161822200.png)

<center>PIC</center>


## 设计与实现



![image-20191030200004952](/images/2019-11-06/image-20191030200004952.png)



### 1) Preprocessing

加载需要再编译的节，加载符号表和重定位节，采用线性扫描算法反汇编，还原CFG



### 2) Symbolization

Symbolization是RetroWrite改写二进制文件的核心步骤，RetroWrite通过重定位信息和恢复的CFG来标识text节和data节中的常量，并将它们转成label

- Control Flow Symbolization

  将jump、call的引用符号化，code-to-code

- PC-relative Addressing

  因为x86_64 PIC代码使用rip相对寻址，计算rip相对地址的操作数，为其标记label。这就包含了间接跳转和间接函数调用。并以此修改CFG。code-to-code、code-to-data

- Data Relocations

  模仿动态加载器处理外部引用的重定位信息(extern dynamic_symbol_table)，并将重定位指向的位置标记label。处理了data-to-data、data-to-code的引用



### 3) Instrumentation Passes

插桩，以使用AFL或ASan



### 4) Instrumentation Optimization

分析插桩的代码是否会影响正在使用的寄存器的状态。选择性的保存寄存器状态或为插桩代码分配空闲的寄存器

需要三个条件：

- 插桩的pass在module、function、basic block的粒度上进行操作，需要标记function、basic block，RetroWrite通过符号表来标识函数。

- 确保插桩代码不会破坏二进制文件的ABI

  x86_64 System V 的ABI规定叶函数（不会调用其他函数）可能会使用rsp至rsp-128作为栈，而不需要显式声明。所以对于叶函数插桩时，要使用单独的栈来保存和恢复状态。

- 自动分配寄存器给插桩的代码使用

  插桩代码需要保证程序的寄存器、标志寄存器等不受影响，最保险的办法是将所有状态保存起来。但为了避免频繁保存状态带来很大的overhead，RetroWrite会对寄存器进行liveness的分析，优先分配non-live的寄存器，使用live的寄存器时则会在插桩前后保存和恢复寄存器状态。如果插桩代码会破坏标志寄存器，则RetroWrite会检测出来并对其进行保存和恢复。



### 5) Reassembly

使用现成工具编译成二进制文件



## Binary Address Sanitizer

ASan是通过在编译时插桩，在分配的内存前后设置redzone，用shadow byte来标记对应内存的访问情况，并通过检查shadow byte的标记来判断是否访问非法内存。

作者实现了一个基于retrowrite的Binary Address Sanitizer。但由于变量类型、数组长度等信息缺失，为了避免误报，ASan-retrowrite只以栈帧为粒度检测局部变量内存越界的问题。

同样的，对于全局变量，由于编译完后数据布局、语义信息丢失，为了避免误报，作者放弃了对全局变量的内存越界检测。

![image-20191105163114546](/images/2019-11-06/image-20191105163114546.png)

![image-20191031135641515](/images/2019-11-06/image-20191031135641515.png)



## Evaluation

作者针对RetroWrite的可扩展性、ASan-retrowrite的性能以及覆盖率、afl-retrowrite进行了评估。

### 可扩展性

作者选取了多个软件证明其二进制改写技术具备可扩展性。可以改写任意由C语言编译而来的二进制文件

![image-20191031141142977](/images/2019-11-06/image-20191031141142977.png)

### ASan-retrowrite性能评估

比Valgrind快300%，比ASan慢65%

![image-20191031141552843](/images/2019-11-06/image-20191031141552843.png)

### ASan-retrowrite覆盖率评估

使用Juliet测试集

![image-20191031142115516](/images/2019-11-06/image-20191031142115516.png)

### afl-retrowrite评估

使用LAVA-M以及real-world中的软件测试

- CF: Source code instrumentation at LLVM-IR level, through afl-clang-fast,
- G: Source code instrumentation at assembly level, through afl-gcc,
- Q: Runtime instrumentation through afl-qemu
- DI: Static rewriting through trampolines, through afl-dyninst
- RW: Static rewriting through afl-retrowrite (our solution).

![image-20191031155520682](/images/2019-11-06/image-20191031155520682.png)

![image-20191031155533197](/images/2019-11-06/image-20191031155533197.png)

![image-20191106094817283](/images/2019-11-06/image-20191106094817283.png)

## 结论

实现了一个能够保证完整性的、针对x86_64、PIC的二进制文件改写框架， RetroWrite。且基于RetroWrite开发了能够与Asan兼容的ASan-retrowrite和能收集覆盖率的binary-AFL。

ASan-retrowrite性能优于目前最好的binary-only memory checker, Valgrind memcheck，且能与ASan兼容。

作者的binary-AFL与source-based AFL性能接近，远优于目前基于QEMU的黑盒二进制文件模糊测试。



## Test



```
c
#include <stdio.h>

int global_var = 0;

int main(){
        printf("%d\n", global_var);
}
```



```
asm
.section .rodata
.align 4
.type	_IO_stdin_used_6f0,@object
.globl _IO_stdin_used_6f0
_IO_stdin_used_6f0: # 6f0 -- 6f4
.LC6f0:
	.byte 0x1
.LC6f1:
	.byte 0x0
.LC6f2:
	.byte 0x2
.LC6f3:
	.byte 0x0
.LC6f4:
	.byte 0x25
.LC6f5:
	.byte 0x64
.LC6f6:
	.byte 0xa
.LC6f7:
	.byte 0x0

.section .data
.align 8
.LC201000:
	.byte 0x0
.LC201001:
	.byte 0x0
.LC201002:
	.byte 0x0
.LC201003:
	.byte 0x0
.LC201004:
	.byte 0x0
.LC201005:
	.byte 0x0
.LC201006:
	.byte 0x0
.LC201007:
	.byte 0x0
.LC201008:
	.quad .LC201008
.section .bss
.align 4
.type	completed.7697_201010,@object
.globl completed.7697_201010
completed.7697_201010: # 201010 -- 201011
.LC201010:
	.byte 0x0
.LC201011:
	.byte 0x0
.LC201012:
	.byte 0x0
.LC201013:
	.byte 0x0
.type	global_var_201014,@object
.globl global_var_201014
global_var_201014: # 201014 -- 201018
.LC201014:
	.byte 0x0
.LC201015:
	.byte 0x0
.LC201016:
	.byte 0x0
.LC201017:
	.byte 0x0
.section .text
.align 16
	.text
.globl main
.type main, @function
main:
.L64a:
.LC64a:
	pushq %rbp
.LC64b:
	movq %rsp, %rbp
.LC64e:
	movl .LC201014(%rip), %eax
.LC654:
	movl %eax, %esi
.LC656:
	leaq .LC6f4(%rip), %rdi
.LC65d:
	movl $0, %eax
.LC662:
	callq printf@PLT
.LC667:
	movl $0, %eax
.LC66c:
	popq %rbp
.LC66d:
	retq
.size main,.-main
```

可以正常汇编成二进制文件并执行。



## Future work

- 针对RetroWrite的limitation：

  - x86_64
  - PIC
  - C
  - Not strip

- 当我们拥有可以修改的Reassembleable Assembly后，可以做哪些工作？

  RetroWrite的应用场景：对无源码的binary做大量的修改

  - 对二进制文件进行污点分析，MPK插桩保护
  - CTF AWD中的patch
  - ...
