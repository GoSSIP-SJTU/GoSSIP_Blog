---
layout: post
title: "ConFIRM: Evaluating Compatibility and Relevance of Control-flow Integrity Protections for Modern Software"
date: 2020-03-27 18:14:55 +0800
comments: true
categories: 
---

> 会议：USENIX'19
>
> 作者：Xiaoyang Xu, Masoud Ghaffarinia, Wenhao Wang, Kevin W. Hamlen, Zhiqiang Lin
>
> 单位：University of Texas at Dallas; Ohio State University;
>
> 链接：[原文](https://www.usenix.org/conference/usenixsecurity19/presentation/xu-xiaoyang) | [代码](https://github.com/SoftwareLanguagesSecurityLab/ConFIRM)

对于CFI方案的评估，性能和安全性考虑得比较多，但软件兼容性考虑得比较少。实际上，一般用来做实验的都是coreutils和SPEC这样的程序，并没有很广泛的代表意义。作者提出了ConFIRM作为一套CFI的评估标准，结果发现已有CFI算法下，约有47%的CFI相关代码特性存在兼容问题或保护效果被削弱。作者还讨论了CFI的未来发展。

<!-- more -->

## 背景

控制流劫持攻击。攻击者通过数据攻击（栈溢出等）覆写函数指针、函数指针等，从而改写控制流。CFI即通过静态分析确定每个跳转点可能的所有目标，并在运行时加以限制，由此避免控制流劫持。

2015~2018年期间四大会上发表了25种新的CFI方法，改进主要面向性能和准确性、安全性，实验多采用SPEC套件和概念验证代码。但事实上用户程序里可能有很多特性，如LLVM、GCC、MSVC都为了保证使用了返回地址自省（函数读取返回地址作为参数）的代码正常运行，其CFI方案不保护返回地址。

ConFIRM目前包括24组测试，每个测试都代表一种可能造成CFI困难的代码特性或习语。作者用ConFIRM测试了12种CFI实现，47%的case种出现了不兼容或是保护失败。作者还发现了一些SPEC套件没有揭示的性能问题。

## 特性列表

![image-20200309104824862](/images/2020-03-27/image-20200309104824862.png)

### Indirect Calls

- 函数指针；
- 回调：函数指针的一种特例，因为回调函数的使用者的代码可能不能修改（如OS）
- 动态链接：可能以编程方式加载目标函数；此外动态库可能会导出数据指针
- 虚函数：对象调用虚函数时查虚函数表，因此虚函数表的完整性需要被保护；但攻击者可以利用错误但合法的虚函数表（COOP，伪造OOP）；另外COM等反射接口需要动态构造虚函数表。
- 尾调用优化：编译器可能会将嵌套的尾调用优化成并排的先后调用，这样静态分析的调用关系就与动态分析不相符。（疑问，这样的callee本身是不显然的，直接当成一个整体应该就好了）
- switch语句：有时switch语句也是通过查表实现的，但此时的target不是函数，而是基本块。
- return语句：return的跳转地址一定存在栈上，更难与数据作区分；source-free工具很难替换ret指令，因为太短了；source-aware方案替换ret指令则容易引起很大的性能开销；
- 异常处理（setjmp等）可能引起call/return不匹配。

### 其他问题

- 多线程问题。前面讲到不过不使用ret指令，性能惩罚会很大，所以一种方法是生成guarded return，也就是在返回前检查返回地址。但在多线程情况下，这是一个TOCTOU攻击窗口，作者将其称为cross-thread stack-smashing attack。这个问题被发现存在于所有体系结构下的所有CFI方案中。

- PIC问题。PIC需要先获取PC，然后加偏移，但是source-free CFI可能会改变偏移量；只有识别出PC获取和偏移计算过程才能正常工作，但获取PC的方法还是非常多的，包括之前讲到的返回地址自省。

- C++式的异常处理

  C++的异常处理也是需要查表的，不过这个表一般是只读的。

  但Windows给C语言也添加了SEH的支持，exception node储存在栈上，exception head储存在OS中，因此栈覆写同样会导致危险，且使用处在外部模块。

  XP往后支持VEH，不使用链表，不储存在栈上，但是储存在堆上，且完整性保护机制形同虚设（可逆），且handler的注册仍然涉及到跨模块的调用。

- 调用约定。问题在于CFI方案可能会挪用一些不常用的寄存器，因此在一些情况下与正常的代码向冲突；另一种可能的情况是，编译器可能会生成一些非标准调用约定的代码。

- TLS回调：TLS区可以注册初始化或销毁回调。回调函数表会是只读的，所以不需要额外保护，但source-free方案需要允许调用。

- 页管理函数。如`mmap`和`mprotect`，可能引起权限问题，但是有非常广泛的合法用途。

- 动态代码生成。可以分为3类。

  JIT编译器。JIT现在已经是很常见的脚本语言的优化手段了，而实际上很多软件内部都有脚本组件，如浏览器等。

  自解压软件。安装器或者是一些移动设备上的程序，减小程序体积是用户乐于看到的。

  钩子。不同于回调，钩子主要是修改函数首部，在函数开始时跳到挂钩函数。很多Windows程序里都有这样的操作。

### 测试样例表

测试的目标包括：能拦截恶意行为；不影响良性行为；可以设置循环次数以作性能测试。

| 类别         | 项目              | 说明                                   |
| ------------ | ----------------- | -------------------------------------- |
| 函数指针     | fptr.             |                                        |
|              | callback.         | callsite可能在kernel中                 |
| 动态链接     | load_time_dynlnk. |                                        |
|              | run_time_dynlnk.  |                                        |
|              | delay_load.       | Windows only;                          |
|              | data_symbl.       |                                        |
| 虚函数       | vtbl_call.        |                                        |
|              | code_coop.        | GTK+/COM；伪造的虚函数表；             |
| 尾调用       | tail_call.        |                                        |
| switch       | switch.           |                                        |
| return       | ret.              |                                        |
| 异常处理     | unmatched_pair.   | setjmp/longjmp                         |
|              | signal.           |                                        |
|              | cppeh.            |                                        |
|              | seh.              | Windows only;                          |
|              | veh.              | Windows only;                          |
| 调用约定     | convention.       | x86/x64                                |
| 多线程       | multithreading.   | stack smashing                         |
|              | tls_callback.     | Windows source-free only; 不应被阻止； |
| PIC          | pic.              |                                        |
| 页API        | mem.              |                                        |
| 动态代码生成 | jit.              |                                        |
|              | api_hook.         | Windows only;                          |
|              | unpacking.        | source-free only;                      |

## 评估

下表显示了12种CFI方案在24个测试中的表现。百分比或勾表示能有效保护，百分比数字为性能overhead，三角叹号表示能运行但没有有效保护，叉表示没有成功运行，N/A表示该测试例不适用。

![image-20200309171811660](/images/2020-03-27/image-20200309171811660.png)

对于Windows上运行的LLVM CFI，不支持编程方式的动态链接库加载，不检查传递到外部模块的回调指针，不支持ret指令。LLVM ShadowStack则补充了ret指令的支持，但在call/ret不匹配的情况下不能正常工作。

MCFG为MSVC所使用的方案，与LLVM相似，不能处理外部模块回调、不支持ret指令，但对动态链接的支持明显更好。有趣的是微软自己的方案也不支持VEH。

Reins和OFI是Windows上的source-free方案。Reins支持了回调处理但仍不能通过COOP测试，并支持ret指令，包括不匹配的情况。OFI与Reins相比提供了更全面的动态链接支持、异常处理支持，并通过了COOP测试、TLS回调测试。不过这两个source-free方案都不能保护PIC代码。

GCC-VTV只是一个vtable保护方案，GCC中没有其他的CFI支持。VTV并不能防御COOP攻击，不过它好在不会break任何特性。其实LLVM也做到了。Linux上的LLVM测试结果和Windows上差不多，除了动态链接的支持要完整一些。

MCFI和$\pi$CFI是source-aware方案。$\pi$CFI有一个关闭尾调用优化的选项，可以提高精度，代价是一定的性能overhead，但这个选项没有带来兼容性上的改变。两者在回调、动态库加载方面的兼容性都有比较大的问题，而对ret指令、异常处理的支持较好。

PathArmor是一种上下文相关的方案，通过在内核态中读取跳转记录来进行判断，以此来对系统API进行保护。是唯一一个能处理mem类型的方案，但是伴随着惊人的高overhead。

Lockdown是一个需要从二进制中读取符号信息的方案，因此对跨模块调用的支持较好（1.45%的动态链接开销），但其他功能的性能开销还是较大。

----

测试中没有一个方案可以防御跨线程stack smashing，而且事件中这样的时间差攻击只需要几秒钟就能成功。

页API也应该是重要的保护目标，但是只有PathArmor成功了，且有很高的代价。

动态代码生成是另一个兼容性重灾区。目前面向动态代码生成的CFI方案是有的，不过还需要相当的手工工作。

![image-20200309204413152](/images/2020-03-27/image-20200309204413152.png)

总的来看，几个编译器内置的方案都至少不会导致运行失败；OFI给出了最好的有效保护率。

----

![image-20200309205544754](/images/2020-03-27/image-20200309205544754.png)

从性能上的相关性来看，尽管有个别的测试有相对较高的相关度，整体上SPEC套件并不能表现很多的CFI特性。

