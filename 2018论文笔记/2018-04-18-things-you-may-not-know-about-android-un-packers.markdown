---
layout: post
title: "Things You May Not Know About Android (Un)Packers"
date: 2018-04-18 18:42:15 +0800
comments: true
categories: research_paper
---


作者：Yue Duan,Mu Zhang,Abhishek Vasisht Bhaskar,Heng Yin,Xiaorui Pan,Tongxin Li,Xueqiang Wang,XiaoFeng Wang 

单位：University of California, Riverside † Cornell University ,Grammatech. Inc.,Indiana University Bloomington,Peking University

链接：http://wp.internetsociety.org/ndss/wp-content/uploads/sites/25/2018/02/ndss2018_04A-4_Duan_paper.pdf



## 概述

DROID UNPACK是一个基于 DroidScope模拟器实现的一个脱壳工具。

![1](/images/2018-04-18/1.png)

整个Android系统以及被加壳的APP运行在模拟器中，分析与脱壳工作实现在模拟器外。

DROID UNPACK实现了4个工具来分析被加壳的APP：

- 隐藏代码提取器：用于识别并获取包含DEX、OAT方法的内存区。
- 多层次脱壳检测器：大多数保护壳并不会一次性把保护程序释放到内存中，而是经过多次操作慢慢释放。多层次脱壳检测器可以记录这种逐步释放保护代码的行为。（The Multi-layer Unpacking Detector discovers iterative unpacking operations that intermittently occur in multiple layers.）
- 代码自修改检测器：用于检测一种更隐蔽的脱壳操作，即故意删除先前执行过的代码。
- JNI调用监视器：用于监控来自JNI接口的敏感API调用。

<!--more-->

## 隐藏代码提取器

```
1: procedure LOCATE CODE IN MEM (pc)
2: 	mod ← GET CURRENT MODULE (pc)
3: 	if mod == “libart.so” then
4: 		func ← GET CURRENT FUNCTION (pc)
5: 		if func == “DoCall(ArtMethod*)” then
6: 			md ← GET DEX METH (ArtMethod∗)
7: 		else if func == “ArtMethod::Invoke()” then
8: 			md ← GET NATIVE METH (ArtMethod∗)
9: 		end if
10: 	mem method ← GET ADDRESS RANGE (md)
11: 	return mem method
12: end if
13: return ∅
14: end procedure
```

通过检测程序寄存器PC的值来确定现在运行的函数是否是`ArtMethod::Invoke()` 或者`DoCall()` ，如果是，则记录马上要调用的Java method。

![2](/images/2018-04-18/2.png)

同时，拦截所有内存写操作，并且记录所有被修改的内存，保存在MemUD dirty 中。如果即将要调用的method在刚刚被修改过的内存中，则表示壳释放的代码被检测到，记录所有被壳还原的method代码。

##代码自修改检测器

记录所有被执行过的内存区域地址，并保存在Memcode中，若检测到之后某次执行的内存地址被修改过，且保存在Memcode中，则表示检测到代码自修改情况。

## 多层次脱壳检测器

1. 使用另一个集合MemUL dirty记录被修改过的所以内存区域
2. 如果即将要执行的内存区域包含在MemUL dirty中，则表示壳程序已经释放了部分代码，记录这次释放记录，并清空MemUL dirty。
3. 重复1、2操作，直到程序结束，即可得到本次程序运行过程中壳释放了几次原始代码。

## JNI调用监视器

通过监控Java层与native层的上下文转换，DROID UNPACK可以检测到所有的JNI调用的入口及退出点。同时使用PScout来识别从native层对一些敏感的Android API调用。


## 还原被加密的Java method

1. hook `ArtMethod::Invoke()` in `libart.so` ，追溯到其相应的`DexFile` 并还原所有的`OatMethod`
2. hook`DoCall()` in`libart.so` ，追溯到其相应的`DexFile` 并还原所有的`DexMethod`

## 该工具的缺陷

1. 只能dump被执行过的代码
2. 不能抵抗模拟器检测
3. 无法脱VMP壳

## 关于加壳与脱壳技术的现状

1. 加壳技术被恶意软件广泛滥用，70%的加壳恶意软件使用定制的加壳技术。

2. 加壳技术发展迅速，虚拟机保护技术将是未来的重点发展方向。

3. ![3](/images/2018-04-18/3.png)

   部分商用加壳工具所支持的保护技术以及存在的缺陷

   Context switching via JNI ： 通过JNI调用切换上下文

   Native/DEX obfuscation ： 字节码与native代码的混淆

   Multi-layer unpacking ： 壳程序不一次性解密所有代码到内存中，而是在运行中逐步解密。

   Pre-compilation ：将函数的字节码转换成native code

   libc.so function hooking ： hook `libc.so`中一些关键函数，如`read` `write` `open` ，如果调用了这些函数则程序会自动退出。

   Component hijacking vulnerability ： 组件劫持漏洞，360加固程序存在该漏洞，通过该漏洞可以从远程服务器下载一个文件并替换该APP目录内任意文件。

   Information leakage ： 腾讯加固会收集设备ID、MAC地址等敏感数据，并且通过HTTP协议传输到它自己的服务器内。
