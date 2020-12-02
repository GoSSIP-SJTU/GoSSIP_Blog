---
layout: post
title: "SoK: Make JIT-Spray Great Again"
date: 2018-09-25 13:08:23 +0800
comments: true
categories: 
---

原文：https://www.usenix.org/system/files/conference/woot18/woot18-paper-gawlik.pdf

作者：Robert Gawlik、Thorsten Holz

出处：WOOT 2018

## 1. 论文简介

JIT可以加快脚本代码的执行。在浏览器中通常会使用JIT。JIT编译器对于攻击者来说是一个很大的攻击面，存在各种各样的攻击，例如：
1. JIT-Spray
2. JIT-based code-reuse attacks
  另外JIT编译器由于代码很复杂，也导致了攻击面很大。

作者的贡献：
1. 对JIT-Spray在x86和ARM上的使用进行了调查，包括了学术界和非学术界的一些攻击案例，并介绍了攻击技术。最近的JIT-Spray攻击技术出现在Mozilla Firefox的ASM.JS上
2. 将JIT-Spray和JIT-based code-reuse进行区分，解释了基于JIT编译器的代码重用攻击
3. 总结了一些利用JIT编译器绕过缓解措施的方法，并介绍了一些这些年的JIT的一些防护措施

<!--more-->

## 2. 基础知识

JIT-Spray：如果一个高级语言的表达式中含有有固定值，并且这个高级语言是JIT编译的,那就可以在运行时嵌入恶意代码，从而绕过DEP。如果如果攻击者尝试去分配（Spray）很多的这类代码，代码的地址就变得可以预知了，从而绕过ASLR。最后需要获得程序的控制流，这种情况下，JIT编译器的UAF、Type Confusion、堆溢出就很有用了。

1. Mozilla Firefox使用的JIT编译器是IonMonkey
2. Google Chrome的V8使用的JIT编译器是TurboFan
3. JVM使用的JIT编译器是Oracle HotSpot
4. Microsoft dotNet框架使用的RyuJIT
5. Lua使用的LuaJIT

最流行的对JIT编译器的攻击方法是JIT-Spray，主要针对Adobe Flash的ActionScript Virtual Machine

## 3. JIT-Spray

对于x86架构而言，如果攻击者已经对JIT编译器有了足够的控制，并控制JIT编译器去喷射一些含有立即数的指令。

作者在这里介绍了3个案例，分别是ActionScript上的JIT-Sprit（X86），ASM.JS，以及ARM上的JIT-Spray.

## 4. JIT-Based Code Reuse

在一些有代码重用保护的情况，对程序里的静态代码进行重用就变得不可行了，但是如果所使用的gadget是来自JIT编译器喷射的，那代码重用攻击又变得可行了。但是对于代码重用攻击来说，需要提前进行信息泄露，泄露出地址信息。

## 5. Abusing JIT-Compiler Flaws

前面的方法大多是利用了高级语言代码在被JIT编译器编译成native code时直接翻译过去的常量来作为攻击时执行的shellcode，并绕过一些缓解措施，这个过程中主要还是出在JIT编译器的工作原理。

在这里，作者又了一些更多的利用JIT编译器设计的瑕疵来绕过一些缓解措施的方法。例如对于DEP来说，现在的JIT编译器在工作时通常是先让某个代码区域可写不可执行，然后往里面写入JIT生成native code，然后再让其变成可执行不可写。但是如果利用这个时间窗口，在这个代码区域还是可写不可执行的时候，写入自己想要的shellcode（通过其它的线程），就能绕过DEP了。

## 6. Mitigations

对于利用JIT编译时高级语言的常量作为payload来进行攻击，相应的缓解措施有constant folding和constant blinding，constant blinding就是将常量都先用一个密钥进行异或，在相应代码运行时再将其还原为原来的值。

## 7. 总结

如下图所示是作者以图表形式对JIT相关攻击防护措施的总结。

![](/images/2018-09-25/15364909444915.jpg)

作者认为基于JIT编译器的攻击和防御还会进行下去，因为很多缓解措施都还不完善。
