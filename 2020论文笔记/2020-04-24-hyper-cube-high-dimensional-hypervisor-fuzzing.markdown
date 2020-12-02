---
layout: post
title: "HYPER-CUBE: High-Dimensional Hypervisor Fuzzing"
date: 2020-04-28 18:56:45 +0800
comments: true
categories: 
---
> 作者：Sergej Schumilo, Cornelius Aschermann, Ali Abbasi, Simon Wo ̈rner and Thorsten Holz
> 
> 单位：Ruhr-Universita ̈t Bochum
> 
> 会议：NDSS 2020
> 
> [HYPER-CUBE: High-Dimensional Hypervisor Fuzzing](https://www.ndss-symposium.org/wp-content/uploads/2020/02/23096.pdf)


---

## Abstract

虚拟机监视器（Virtual machine monitors, VMMs, also called Hypervisor）是现代软件栈的重要组成部分，攻击者若能攻陷虚拟机监视器将对云提供商、云基础设施造成严重的安全威胁。

不同于一维fuzzer（AFL），HYPER-CUBE可以与任意数量的接口进行交互，故称为High-Dimensional fuzzer。

作者通过评估表明，HYPER-CUBE与最新的虚拟机监视器fuzzer相比，能发现更多的bug、有更高的代码覆盖率。HYPER-CUBE发现了54个新的bug，申请了37个CVE。

作者的贡献：

- 设计了一种多维度的、平台独立的、针对不同接口的模糊测试方法
- 设计了一种针对于hypervisor的、高效的模糊测试方法。且与被测试的hypervisor独立。

- 在一个定制的OS中实现了上述方法，称该OS为HYPER-CUBE。并证明HYPER-CUBE能够发现现实世界hypervisor中的漏洞。

代码将开源于[https://github.com/RUB-SysSec/hypercube](https://github.com/RUB-SysSec/hypercube)（暂时无法查看）

<!-- more -->

## BACKGROUND

#### *x86 Boot Process*

在x86机器上的Boot Process第一个阶段是启动*Basic Input/Output System* (BIOS) /*Unified Extensible Firmware Interface* (UEFI)。它们的工作主要是测试和初始化CPU、内存等硬件。接下来会执行OS的bootloader，bootloader会将加载并执行OS kernel代码，OS kernel随后会对剩下的硬件进行配置。（interrupt controllers, paging, or PCI devices）



#### *Input/Output on x86*

Intel x86处理器实现了多种与设备的交互方式。如下图所示

![image-20200409230029793](/images/2020-04-24/image-20200409230029793.png)

- *Port I/O*，只能通过in out指令访问
- *Memory-Mapped I/O*（MMIO），由于使用更加灵活、支持更多的带宽，现代硬件更多的使用MMIO，而非port I/O。
- *Direct Memory Access*（DMA），允许硬件直接读写内存，而不需要CPU介入。适用于传输大量数据。



#### *Hypervisor*

一般来说，VMM中的VMs是没有办法控制和访问（除了分配给它的）物理硬件设备。hypervisor会提供virtual CPUs（vCPU）和虚拟化物理内存并模拟必要的设备。当VM需要进行特权操作时，比如访问物理硬件设备，会造成一次VM-exit。VM-exit会将控制权转给hypervisor，hypervisor模拟进行该特权操作。

##### *CPU and Memory Virtualization*

最原始的hypervisor依赖于二进制转译技术（binary translation），需要完全虚拟化CPU和内存。即对于所有的指令都需要进行翻译，并对虚拟化CPU和虚拟化内存进行相应的操作。这样模拟执行特权指令可以保证VM不会脱离虚拟化上下文。

但这样的虚拟化缺乏硬件支持，存在很大的开销。因此，目前主流的CPU厂商都进行了针对虚拟化的硬件扩展，Intel的“Intel VT-x”，AMD的“AMD-V”。Intel VT-x的实现方式为：增加了一个比OS kernel权限更高的执行模式（Ring -1，root mode），hypervisor运行在root mode下，拦截一些特定的指令/事件，从而限制guest kernel访问真实硬件，而不需要模拟特定指令。



##### *Full-Virtualization*

全虚拟化，使用虚拟机协调guest操作系统和原始硬件，VMM在guest操作系统和裸硬件之间用于工作协调，一些受保护指令必须由Hypervisor（虚拟机管理程序）来捕获处理。全虚拟化的运行速度要快于硬件模拟，但是性能方面不如裸机。

##### *Para-Virtualization*

半虚拟化是另一种类似于全虚拟化的技术，它真实的硬件，但是它的guest操作系统集成了虚拟化方面的代码。引入了hypercall指令，类似于syscall，可以主动VM-exit。

半虚拟化需要guest操作系统做一些修改，使guest操作系统知道自己是处于虚拟化环境的，但是半虚拟化提供了与原操作系统相近的性能。



##### *Device Emulation* 

为了能够启动OS，hypervisor需要模拟标准设备，如中断控制器、时钟及其他外围设备。为了与硬件通信，需要模拟MMIO和Port I/O

![image-20200423160435011](/images/2020-04-24/image-20200423160435011.png)

目前的hypervisor仅支持模拟一部分的标准设备，但模拟设备的代码量很大，例如QEMU 4.0存在超过400k行用于模拟设备的c代码，是一个值得关注的攻击面。



#### *Fuzzing Hypervisor*

不同于典型的fuzzer，Fuzzing hypervisor的挑战：

- 大量用于交互的接口
- guest system能够触发hyper call、IO操作以及读写MMIO区域。
- 重启hypervisor比重启user-space process开销昂贵很多。

所以目前很多的 hypervisor fuzzer针对单独的接口，并为其提供隔离的环境，以避免重启VMM。然而将众多接口隔离测试并不能十分有效的发现问题，经过作者的实验表明，与不同的接口进行一系列的交互可能发现漏洞。

hypervisor中出现漏洞的影响更为严重，轻则DOS，重则虚拟机逃逸。



#### *Related Work*

目前针对fuzzing hypervisor的研究更多的是在工业界，2016年balckhat上提出了能够针对hypervisor进行模糊测试的[AFL插件](https://www.blackhat.com/docs/eu-16/materials/eu-16-Li-When-Virtualization-Encounters-AFL-A-Portable-Virtual-Device-Fuzzing-Framework-With-AFL-wp.pdf)，他们实现了一个扩展BIOS，能够与外界通信并针对Virtual Device进行模糊测试，并能够收集覆盖率信息。其架构如下图所示。

![image-20200423135409945](/images/2020-04-24/image-20200423135409945.png)

而学术界针对于 hypervisor fuzzing的研究此前仅有在2017年RAID提出的[VDF](https://www.cs.ucr.edu/~heng/pubs/VDF_raid17.pdf)。然而VDF的作者从QEMU代码库中手动将虚拟设备提取出来后，使用AFL进行fuzz。该方法有明显的缺点：需要手动提取设备模拟器。

![image-20200423142304624](/images/2020-04-24/image-20200423142304624.png)



## Threat Model

- 攻击者拥有虚拟机中的完全控制权限
- 攻击者的目标是控制其他虚拟机或host机。



## Design & Implementation

![image-20200409232134858](/images/2020-04-24/image-20200409232134858.png)

作者的fuzzer主要由三部分组成：

1. 作者定制的OS，称为HYPER-CUBE OS。其在目标hypervisor中启动并枚举存在的硬件接口。作者实现HYPER-CUBE OS的目的是能够完全掌握fuzz的过程。
2. Tesseract，字节码解释器。它通过解释字节码来对hypervisor进行模糊测试。
3. 一系列外部的工具，能够为Tesseract解释器提供字节码、反编译字节码成c代码等。



作者设计的结构能够达到三个方面的目标：

1. High performance fuzzing。

   一方面，如果使用Linux或者Windows这种COTS OS，重新启动需要大量的时间。HYPER-CUBE OS十分精简，不与其他中断进行交互，也没有任何硬件初始化程序，而且HYPER-CUBE OS只需要使用很少的内存。这样使得在每一次crash后都能够快速的重启，从而显著提高fuzz的性能。

   另一方面，内置的Tesseract输入字节码进行转译并与hypervisor交互，所有的指令都在最大程度上设计为对fuzzing有帮助的指令且没有非法指令，这样方便随机数发生器的生成字节码；内存地址使用region、offset，而非使用指针。

2. Generic High-Dimensional Fuzzing，先前的hypervisor fuzzer只能针对一到两个特定的接口，VDF仅关注于 MMIO，作者通过Tesseract解释器可以与多个接口进行交互。通过更复杂的交互能够揭露更多的漏洞。

3. Stable and Deterministic Fuzzing，作者使用的HYPER-CUBE OS使得可以完全掌握fuzz的环节，从而提高测试的鲁棒性和确定性。



#### HYPER-CUBE OS

HYPER-CUBE OS出了为fuzzer提供一个基本的环境之外，还有两项主要的工作：内存管理与设备枚举。

内存管理：HYPER-CUBE OS维护了一个小的堆用于动态内存分配；有的交互需要直接访问物理地址，HYPER-CUBE OS也为部分虚拟内存于物理内存做了直接映射。

设备枚举：需要为Tesseract枚举出能够进行fuzz的接口



#### Tesseract

Tesseract的功能是解析字节码并调用对应的处理函数进行I/O操作。字节码能够从外部输入也可以通过内部的伪随机数发生器生成。

Tesseract中的内存使用 (region-id, offset)来表示，使用时region-id和offset都会模上可用的长度。

作者实现了如下opcode handler：

- write mmio(region id, offset, data)
- read mmio(region id, offset)
- xor mmio(region id, offset, mask)
- bruteforce mmio(region id, offset, data, num)
- memset mmio(region id, offset, data, num)
- writes mmio(region id, offset, data, num)
- reads mmio(region id, offset, num)
- mmio write scratch ptr(region id, offset, scratch- id, scratch-offset)  向mmio内存中写入指针
- \*_io() Port I/O相关
- write msr(msr num, mask)
- hypercall(eax, ebx, ecx, edx, esi)
- vmport(ecx, ebx)

作者实现了两种提供字节码指令的方式：PRNG、将字节码字符串嵌入OS镜像中



#### External Tools

最后一个组件是运行在host机上的三个辅助工具：

- logger会记录相关信息。
- minimization tool能够记录下的fuzz操作缩短。
- decompiler工具能够将字节码翻译成等价的c语言代码。



## Evaluation

实验环境：Intel i7-6700 processor (4 cores) and 24 GB of RAM，Ubuntu 16-04 LTS

实验对象：6 个hypervisors: QEMU/KVM (4.0.1-rc4), Bhyve (12.0-RELEASE), ACRN (29360 Build), VirtualBox (5.1.37_Ubuntu r122592), VMware Fusion (11.0.3) and Parallels Desktop (14.1.3)

作者从发现新漏洞的能力、发现旧漏洞的能力、性能等方面进行了评估。


#### *Finding New Vulnerabilities*

作者报告了54个bug，获得了43个CVE。

![image-20200409224622918](/images/2020-04-24/image-20200409224622918.png)



#### *Rediscovering Known Vulnerabilities*

- old CVEs：作者试图去fuzz VDF发现的漏洞，HYPER-CUBE全部找到。
- 作者对四个漏洞进行了分析。
- 2016年blackhat上提出的方法发现VENOM这个漏洞需要1.5小时

![image-20200409235835253](/images/2020-04-24/image-20200409235835253.png)

#### *Performance*

HYPER-CUBE *vs* VDF

针对相同版本的qemu，HYPER-CUBE额外发现了5个漏洞。

##### *Coverage*

- 在6 cores上跑VDF 60天，1 core跑HYPER-CUBE 10min。

- VDF作者设置了种子，HYPER-CUBE从空白状态开始。
- VDF是使用了coverage feedback以提高性能

![image-20200409235735583](/images/2020-04-24/image-20200409235735583.png)



##### *Interpreter Throughput*

作者使用了16MB的opcode，用反编译器生成c代码，1.8million行。

作者用解释器生成、解释、执行16MB opcode的时间与用gcc使用的时间做对比。

![image-20200409235800099](/images/2020-04-24/image-20200409235800099.png)

#### *Restart After Crash*

![image-20200409235914548](/images/2020-04-24/image-20200409235914548.png)



## Conclusion

本文中，作者提出了一种针对于hypervisor与设备模拟器的模糊测试的方案并予以实现。

HYPER-CUBE在开源、非开源的hypervisor中发现了54个不同的bug。

并通过实验表明HYPER-CUBE优于最新的coverage-guided hypervisor fuzzer。