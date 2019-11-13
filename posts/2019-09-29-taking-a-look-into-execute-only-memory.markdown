---
layout: post
title: "Taking a Look into Execute-Only Memory"
date: 2019-09-29 02:56:11 -0400
comments: true
categories: 
---

作者：Marc Schink, Johannes Obermaier

单位：Fraunhofer Institute AISEC

出处：WOOT 2019

资料：[Paper](https://www.usenix.org/system/files/woot19-paper_schink.pdf)
<hr/>
## 1. Abstract & Introduction

为了保护软件的知识产权，微控制器产商使用了XOM(eXecute-Only Memory)，来阻止未授权的第三方固件在开发时的泄露。XOM允许代码执行，但是不允许代码的读取。作者的安全分析表明，由于共享资源的存在（例如CPU和SRAM），这个技术是不足以保护固件的。作者设计了一个方法来自动恢复出被保护的固件。

作者所作出的贡献：

1. 分析XOM作为保护固件不被泄露的可行性
2. 发现了一些基于ARM Cortex-M的微控制器上XOM实现的瑕疵
3. 一个通用的自动恢复被保护固件的程序
4. 一套实用的评估不同产商的设备遭受固件恢复攻击的方法

<!--more-->
## 2. Execute-Only Memory

XOM可以用于多方开发的情景下，对于一个嵌入式设备的库开发者来说，可以将库所对应的内存设置为XOM，从而不让应用开发者读取和修改里面的内容。

![](/images/2019-09-29/Untitled-e6beac59-4cce-49f4-8b5c-9c16a4497f86.png)

对比TEE可以提供完全隔离的代码执行来说，XOM仅仅提供了隔离的Flash区域，但是共享了所有其他的资源，例如CPU和SRAM，即使XOM的读取限制阻止了固件的直接读取，但是攻击者可以通过共享资源的变化，来恢复所执行的指令。作者将共享的资源称为了系统状态。

恢复的流程如下，执行一条指令就进行一次恢复。

![](/images/2019-09-29/Untitled-0fb68070-6597-4cd1-b13b-7dadcafb4a73.png)

## 3. 指令恢复

`mov r0, #23` 的系统状态变化如下图所示。

![](/images/2019-09-29/Untitled-88079b5c-7e81-4661-946a-580bc0eaaf59.png)

执行完后，读pc寄存器即可知道指令长度为2字节，可以看出它修改了r0寄存器，修改r0寄存器的指令很多，例如add，ldr，但是至少可以排除是跳转指令，

指令恢复的过程：

1. 生成输入状态。
2. 指令执行。
3. 指令推断。
   1. Pre-check：根据一些简单的规则去掉一些不可能的指令。
   2. 枚举所有可能的指令
   3. 将这些可能的指令都跑一遍，看结果是否一致

在这个过程后，仍然没办法精确恢复出目标指令。

### 3.1 Load和Store

load指令的形式通常为 `ldr rt, [rn, #imm]`和`ldr rt, [rn, rm]`

SRAM的内容设置成如下所示，这样每个word都有一个4字节的id，每当一条load指令被执行了，内存中的word就会出现在寄存器里，因此目标指令的寄存器和源地址都可以确定了。store指令类似。

![](/images/2019-09-29/Untitled-97035255-0231-4e51-b578-008c7f139054.png)

输入状态的寄存器设置成如下所示，以确认load和store中的用于寻址的寄存器。

![](/images/2019-09-29/Untitled-2fe2771b-5ed2-49fa-8370-cd41c0602d6a.png)

### 3.2 Push和Pop指令

判断指令是否是push和pop很简单，只要看sp是否修改了即可（br也可以修改sp，但是可以通过pc变化来排除br指令）

![](/images/2019-09-29/Untitled-08c5d4e3-f8af-4ac9-a8de-d36f0cbb04e2.png)

### 3.3 用PC寻址的Load指令

`LDR Rd,[PC, #<imm>]`这种指令是从代码内存里直接获取立即数用，由于XOM的内存没办法读，也不能写数据，因此imm不好确定。但是可以将这个指令与普通的LDR进行区分，只要将寄存器都设置成无效地址即可，如果崩溃了就是普通的LDR，没崩溃就是这种PC寻址的Load指令。

### 3.4 Branch指令

需要考虑到的跳转指令有blx，bx，mov，pop，为了区分这些指令，作者将输入状态的寄存器设置为如下图所示，只要判断sp和lr寄存器的变化即可区分出来。

![](/images/2019-09-29/Untitled-6f4a6230-bd7d-42c5-bb78-fdb04975762e.png)

如下图所示是可以恢复出来的指令。

![](/images/2019-09-29/Untitled-31aa4e0a-d079-4574-a8dc-8a2e91a63f3a.png)

## 4. 具体实现

如下图所示是具体的实现，用OpenOCD去调试设备。作者实验的目标设备为Kinetis KV11和STM32L0

![](/images/2019-09-29/Untitled-f86605e6-4bb2-42c5-a19f-eaf521c85463.png)

用单步调试进行代码恢复的效率如下图所示。

![](/images/2019-09-29/Untitled-e749a62e-0a55-415e-99d5-c18a99f08e49.png)

前面的方法是用单步调试来实现，也可以通过中端驱动的方法完成，因为像STM32F/H7和MSP432P4是不能在保护代码里进行单步调试的。

## 5. 缓解措施

最合适的方法是关闭调试接口，但是对于多方开发的情景下是不合适的，因为别人还要调试。另外一个选择是用TEE而不是XOM来保护IP。

## 6. 设备分析

作者分析了几个XOM实现。

### 6.1 STM32设备

STM所开发的一个XOM实现成为PCROP(Proprietary Code Read-Out Protection)，不同的STM32都使用了这个功能，从低功耗的STM32L0系列到高性能的STM32H7系列。

STM32Lx和STM32F4设备都没有限制任何调试功能，例如保护代码的单步执行。CPU寄存器和SRM的内容都是可以观察到的。STM32F7和STM32H7系列的设备则有考虑到这种情况，在保护代码的执行过程中CPU不会被停住，没办法进行单步调试。

### 6.2 Tiva C和MSP432设备

德州仪器的设备，Tiva C and MSP432E4没有限制保护代码执行时的调试，可以直接单步调试。

MSP432P4设备允许XOM内的load。

### 6.3 Kinetis设备

Kinetis系列是NXP开发的。作者发现XOM里所有的load指令都可以读取XOM里的内存。

## 7. 总结

作者所发现的问题总结如下图所示

![](/images/2019-09-29/Untitled-3183f159-8b2c-4243-9579-694218690406.png)
