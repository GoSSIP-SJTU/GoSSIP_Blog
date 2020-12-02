---
layout: post
title: "Shielding Software From Privileged Side-Channel Attacks"
date: 2018-10-11 16:04:59 +0800
comments: true
categories: 
---

![20180905110724](/images/2018-10-11/20180905110724.png)

## 摘要

商用操作系统内核，例如Windows、Mac OS X、Linux、FreeBSD，容易受到多种安全漏洞的影响，攻击者能够通过这些漏洞控制操作系统内核，获取对于所有的应用数据和系统资源的访问权限。

InkTag、Haven、Virtual Ghost等防护系统能够在操作系统内核被攻击者控制的前提下保护敏感应用数据。然而，这些防护系统依旧没有办法防护恶意操作系统内核实施的侧信道攻击。

文章提出了针对恶意操作系统内核实施的页表侧信道攻击与LLC（Last-Level Cache）侧信道攻击的防护方案，并通过在优化过的Virtual Ghost系统上部署以上防护方案，提出了防护系统Apparition。

<!--more-->

## 1 介绍

商用操作系统的单内核架构虽然具有性能表现优异的优点，但是由于缺乏有效的隔离措施，攻击者能够通过一个单一的漏洞控制整个操作系统内核，窃取损坏系统中的所有数据。

InkTag、Virtual Ghost等软件解决方案与Intel SGX、ARM TrustZone等硬件解决方案虽然能够防止恶意操作系统内核直接读取损坏应用数据，但是恶意操作系统内核依旧能够通过利用共享的硬件资源以及应用代码与操作系统代码的交互，以侧信道攻击的方式窃取敏感应用数据。

防护系统Apparition以Virtual Ghost系统为原型，通过使用Intel MPX（Memory Protection Extensions）技术以及消除串行化指令（Serializing Instructions）的方式对于原始Virtual Ghost系统进行了优化，并在此基础上部署针对恶意操作系统内核实施的页表侧信道攻击与LLC侧信道攻击的防护方案。

文章提出的针对恶意操作系统内核实施的侧信道攻击的防护方案不需要对于现有的处理器架构进行任何修改。Apparition通过防止操作系统内核读取或修改存储应用机密信息的内存页所对应的页表项（Page Table Entry），以防护页表侧信道攻击，并通过采用Intel CAT（Cache Allocation Technology）技术以及防止物理内存共享的技术，以防护LLC侧信道攻击。

## 2 攻击模型

文章的攻击模型假设攻击者控制了操作系统内核，并希望窃取敏感应用数据。应用开发者采取了相应的安全加固措施保证了应用本身的安全性。攻击者无法物理接触设备，且由于Virtual Ghost等防护系统的存在，攻击者无法直接直接读取应用内存数据。

文章重点关注通过软件实施的页表侧信道攻击与LLC侧信道攻击，通过硬件实施的侧信道攻击以及预测执行侧信道攻击不在文章的讨论范围之内。

## 3 侧信道攻击

现代操作系统中进程间共享的系统资源状态是导致侧信道攻击的主要原因。此外，恶意操作系统能够通过控制特权处理器状态等方式创造另外的侧信道攻击方式。

### 3.1 页表侧信道

在存在防护系统的前提下，操作系统内核无法直接读取或修改存储应用私密数据的内存页所对应的页表项，恶意操作系统内核依旧能够通过以下方式利用虚拟地址至物理地址的变换机制推测应用的内存访问模式：

**交换**

恶意操作系统内核通过将特定内存页交换到磁盘上，并监视防护系统何时要求将此页面交换回内存中，以推测应用的内存访问模式。

**读取页表项**

许多处理器会在访问页面数据或向页面写入数据时，在内存页所对应的页表项中设置脏位或访问位。通过持续不断地检查内存页所对应的页表项中的内容，恶意操作系统便能够知道应用何时第一次读写特定的内存页。

**推测地址变换的缓存**

恶意操作系统通过对存储虚拟地址至物理地址变换的Cache，使用PRIME+PROBE与FLUSH+RELOAD的Cache侧信道攻击推测应用的内存访问模式。

### 3.2 Cache侧信道

恶意操作系统通过测量应用的Cache使用模式以推测机密数据。两种常见的Cache侧信道攻击为PRIME+PROBE与FLUSH+RELOAD，这两种侧信道攻击均可用于攻击私有Cache与共享LLC。现有的Cache分隔技术仅能够有效防止非特权攻击者实施的Cache侧信道攻击，而对于特权攻击者，即恶意操作系统内核则无能为力。

### 3.3 指令追踪侧信道

如果应用的指令序列存在对敏感应用数据的控制依赖，则恶意操作系统便可以根据应用对于指令内存的访问模式，推测应用的机密信息。

## 4 Virtual Ghost改进

### 4.1 设计

![20180905143713](/images/2018-10-11/20180905143713.png)

Virtual Ghost是一个基于编译器的虚拟机，以SVA为基础。操作系统内核代码被编译成V-ISA指令的形式，Virtual Ghost虚拟机将V-ISA指令翻译成N-ISA指令运行。Virtual Ghost虚拟机强制要求所有的操作系统代码被编译成V-ISA指令的形式，而应用的代码则可以被编译成V-ISA或N-ISA指令的形式。V-ISA指令包括SVA-Core指令与SVA-OS指令。操作系统内核通过SVA-OS指令对于特权硬件状态进行配置。

![20180905145136](/images/2018-10-11/20180905145136.png)

Virtual Ghost将每个进程的虚拟地址空间划分为4个区域：

**User Memory：**应用和操作系统均可读写，不可执行；

**Kernel Memory：**只有操作系统可以读写，不可执行；

**Ghost Memory：**只有应用可以读写，存放应用私密数据；

**Virtual Ghost VM Memory：**Virtual Ghost存放其自身的数据结构，V-ISA指令至N-ISA指令的翻译程序，应用的N-ISA指令代码段，应用与操作系统均不可访问。

Virtual Ghost使用SFI与CFI技术保证进程虚拟地址空间中以上4个区域之间的相互隔离。进程页表被映射为只读内存，Virtual Ghost允许操作系统内核读取页表信息，并通过SVA-OS指令修改页表信息。

### 4.2 Intel MPX

Intel MPX通过4个边界寄存器保存内存对象地址的上界与下界，并通过边界检查指令检查进程访问的虚拟地址的合法性。

文章使用Intel MPX技术，通过将进程的User Memory与Kernel Memory合并作为一个单独的内存对象对待，并将这个大内存对象的地址上下界作为边界检查条件，以改进实现Virtual Ghost采用的SFI技术。

### 4.3 SVA内部直接映射

直接映射是指将一系列连续的虚拟内存页映射到连续的物理内存地址上，以加速虚拟地址至物理地址的变换过程。Virtual Ghost将页表的直接映射存储在Kernel Memory中，并设置写保护功能。当操作系统内核使用SVA-OS指令请求修改页表信息时，Virtual Ghost临时清除X86 CR0.WP位，以临时关闭页表的写保护功能。然而，清除X86 CR0.WP位是一个串行化操作，开销大性能低。

文章通过将页表的直接映射从Kernel Memory转移到Virtual Ghost VM Memory中，并取消写保护的设置，由Virtual Ghost直接读取或修改页表信息，避免了临时清除X86 CR0.WP位的开销。

## 5 侧信道防护

### 5.1 页表侧信道

防护系统必须保护页表信息的机密性与完整性，以防护页表侧信道攻击。

**页表限制**

1. 防止操作系统内核修改映射Ghost Memory的页表项（Virtual Ghost已实现）；
2. 防止存储Ghost Memory的页表被映射到能被操作系统内核访问的内存区域（Virtual Ghost已实现）；
3. 防止操作系统内核读取映射Ghost Memory的页表项：Apparition回收了操作系统内核对于所有页表内存页的读写权限，并仅允许使用SVA-OS指令请求Apparition读取或修改页表信息。

**交换**

在操作系统内核请求换出内存页时，Apparition随机地选择一个内存页并将其交换到磁盘上；在操作系统内核请求换入内存页时，Apparition将该进程所有被换出的内存页全部换入内存中。

Apparition通过以上措施，防止操作系统内核根据内存页的交换推测应用的内存访问模式。

### 5.2 内存页分配侧信道

Apparition通过对Ghost Memory禁用请求页面调度的方式，防止操作系统内核根据Ghost Memory的内存页分配行为推测应用私密数据。

![20180905165216](/images/2018-10-11/20180905165216.png)

此外，Apparition还重新设计了Ghost Memory的物理内存分配方式，通过每次分配随机数量的内存页，以混淆应用的内存分配大小信息。

### 5.3 代码翻译侧信道

尽管Apparition的页表侧信道防护以及内存页分配侧信道防护消除了针对于指令内存的侧信道攻击，Apparition还必须防护将V-ISA指令翻译成N-ISA指令过程中的侧信道攻击。

Apparition将应用的V-ISA指令全部拷贝至Virtual Ghost VM Memory中，并仅允许使用SVA-OS指令请求Apparition翻译V-ISA指令至N-ISA指令，以防护代码翻译侧信道攻击。

### 5.4 LLC侧信道

**防止内存页共享**

防止应用的Ghost Memory被操作系统内核或其他应用访问（Virtual Ghost已实现）。

**Cache分隔**

Apparition将Intel CAT技术与Virtual Ghost已有的内存保护机制相结合，以防止操作系统内核的LLC侧信道攻击。

Apparition通过Intel CAT技术将LLC分区。内核代码、Apparition VM代码各使用一个分区，每个应用分别使用一个独立的分区。当应用的数目超过能够使用的分区时，Apparition VM控制多个应用复用一个或多个分区，并在进行上下文切换时清空相应的LLC分区。

此外，Apparition VM负责维护LLC的分隔，防止操作系统内核重新配置或禁用Cache分隔功能。

### 5.5 指令追踪侧信道

Virtual Ghost将中断程序的状态存放在Virtual Ghost VM Memory中，仅允许操作系统内核使用SVA-OS指令读取或修改中断程序的状态。此外，结合上文所述的代码翻译侧信道防护措施的部署，Apparition能够防护恶意操作系统内核的指令追踪侧信道攻击。

## 6 对预测执行侧信道的影响

尽管预测执行侧信道并不在文章的讨论范围内，但是Apparition能够通过一些改进，以防护这类攻击中利用Cache侧信道的一些变种。

1. 针对Spectre攻击，Apparition无法防止不可信进程之间的所有物理内存共享，包括指令内存以及用户空间内存，故Apparition无法防止Spectre攻击；
2. 针对预测访问越界数据的Meltdown与Spectre攻击，Apparition能够使用抗预测的SFI插桩以及lfence指令插桩的方式，防止预测执行访问越界数据的指令；
3. 针对Meltdown攻击，Apparition能够令用户空间代码、内核空间代码和Apparition VM代码透明地使用不同组的页表和PCID，防护Meltdown攻击。

## 7 实现

![20180905175933](/images/2018-10-11/20180905175933.png)

## 8 评估

### 8.1 方法论

文章的评估采用以下的benchmark与应用，这些程序高度依赖操作系统内核服务，例如文件系统与网络栈：

1. LMBench
2. OpenSSH客户端（不使用Ghost Memory）
3. Ghosting OpenSSH客户端
4. Ghosting Bzip2
5. Ghosting GnuPG
6. Ghosting RandomAccess
7. Ghosting Clang

文章的评估采用以下配置的FreeBSD SVA内核：

1. VG：原始Virtual Ghost系统；
2. Opt-VG：优化Virtual Ghost系统；
3. Opt-VG-PG：部署了页表侧信道防护的优化Virtual Ghost系统；
4. Opt-VG-LLCPart：部署了LLC侧信道防护的优化Virtual Ghost系统；
5. Apparition：部署了页表侧信道防护与LLC侧信道防护的优化Virtual Ghost系统，即Apparition系统。

### 8.2 Virtual Ghost优化

文章所采用的针对Virtual Ghost系统的优化措施，优化效果较好，如下所示：

**Microbenchmarks**

![20180905181852](/images/2018-10-11/20180905181852.png)

![20180905181907](/images/2018-10-11/20180905181907.png)

![20180905181942](/images/2018-10-11/20180905181942.png)

**应用**

![20180905183032](/images/2018-10-11/20180905183032.png)

![20180905183103](/images/2018-10-11/20180905183103.png)

![20180905183125](/images/2018-10-11/20180905183125.png)

![20180905183246](/images/2018-10-11/20180905183246.png)

### 8.3 页表侧信道防护与LLC侧信道防护

**Ghosting RandomAccess**

**Ghosting Bzip2**

**Ghosting Clang**

![20180905183316](/images/2018-10-11/20180905183316.png)

**Ghosting OpenSSH客户端**

![20180905183338](/images/2018-10-11/20180905183338.png)

**Ghosting GnuPG**

![20180905183612](/images/2018-10-11/20180905183612.png)


