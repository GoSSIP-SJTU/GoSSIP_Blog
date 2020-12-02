---
layout: post
title: "SoK: Understanding the Prevailing Security Vulnerabilities in TrustZone-assisted TEE Systems"
date: 2020-09-08 14:36:25 +0800
comments: true
categories: 
---

> 会议：S&P 2020
> 
> 论文名称：SoK: Understanding the Prevailing Security Vulnerabilities in TrustZone-assisted TEE Systems

本文中，作者对TrustZone-assisted TEE Systems中存在的安全问题进行了研究，整理了目前各种TEE的实现中所存在的Attack Surface，并对出现的漏洞进行了归类、讨论了TrustZone-assisted TEE Systems中的缓解措施。文中对比了Qualcomm, Trustonic, Huawei, Nvidia, and Linaro的TEE实现。

<!-- more -->

## Attacking TEE-enabled Devices

Compromising the TEE kernel：攻击者可以通过两条路线攻陷TEE kernel：

- REE App→Android OS→Secure Monitor→TEE Kernel;
- REE App→TA→TEE Kernel

Compromising the REE kernel: REE App→TA，攻击者在compromised TA中可以map NW物理内存，以修改被Linux kernel分配的内存。

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%201.png](/images/2020-09-08/Untitled%201.png)

## Analyzed TEE Systems

作者研究了数个TEE的实现：Qualcomm, Trustonic, Huawei, Nvidia, and Linaro

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%202.png](/images/2020-09-08/Untitled%202.png)

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%203.png](/images/2020-09-08/Untitled%203.png)

作者总结发现其中三个主要的攻击面：

- Architectural （eg. 没有ASLR）
- Implementation （e.g. 实现上的漏洞，buffer overflow）
- Hardware （e.g. side-channel）

## Architectural Issues

### TEE Attack Surface

- SW drivers run in the TEE kernel space
- Wide interfaces between TEE system subcomponents
- Excessively large TEE TCBs （Trusted Computing Base）TCB较大，其中可能存在潜在的漏洞

### Isolation between Normal and Secure Worlds

- TAs can map physical memory in the NW
- Information leaks to NW through debugging channels

### Memory Protection Mechanisms

- Absent or weak ASLR implementations
- No stack cookies, guard pages, or execution protection

### Trust Bootstrapping

- Lack of software-independent TEE integrity reporting 缺乏远程三方认证
- ill-supported TA revocation 废除掉旧的TA，否则攻击者可能可以加载一个旧版本、存在漏洞的TA，以进行攻击。

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%204.png](/images/2020-09-08/Untitled%204.png)

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%205.png](/images/2020-09-08/Untitled%205.png)

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%206.png](/images/2020-09-08/Untitled%206.png)

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%207.png](/images/2020-09-08/Untitled%207.png)

## Implementation Issues

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%208.png](/images/2020-09-08/Untitled%208.png)

### Validation Bugs

在Secure Monitor, TA, Trusted Kernel, Secure Boot Loader中均发现了认证相关的漏洞。

### 功能性错误

程序员实现与标准存在差异而引发的漏洞

### Extrinsic Bugs

包括并发错误、侧信道

## Hardware Issues

硬件架构：

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%209.png](/images/2020-09-08/Untitled%209.png)

### Architectural Implications

- Attacks through reconﬁgurable hardware components TEE，引入新硬件而带来的攻击面，FPGA可以被重新配置
- Attacks through energy management mechanisms，电源管理相关驱动存在攻击面

### Microarchitectural Side-Channels

- Leaking information through caches，在TrustZone-enabled处理器上，cache是被secure world和normal world共享。目前已有成功从cache中恢复128bit AES密钥。
- Leaking information through branch predictor，现代处理器中有branch target buffer (BTB)用于分支预测，被SW和NW共享，可以通过Prime+Probe leak敏感信息。已有攻击从Qualcomm上的kerstore恢复出256-bit私钥。
- Leaking information using Rowhammer，Rowhammer是一种利用软件引起硬件错误，从而影响DRAM的技术。允许攻击者止痛膏内存读操作，反转物理内存中的bits。当TEE中的RSA private被分配在secure/no-secure内存边界处时，可以在no-secure边界频繁读，造成在secure引入错误，从而corrupt private keys。

## DEFENSES FOR TRUSTZONE-ASSISTED TEES

近年来提出的防御措施：

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%2010.png](/images/2020-09-08/Untitled%2010.png)

### Architectural Defenses

- Multi-isolated environments
- Secure cross-world channels
- Encrypted memory
- Trusted computing primitives

### Implementation Defenses

- Managed code runtimes，在TEE中运行解释型语言，使其在受控范围内运行
- Type-safe programming languages，Rust
- Software veriﬁcation，对程序进行建模、符号执行、形式化验证等方式减少bug

### Hardware Defenses

- Architectural countermeasures
- Microarchitectural countermeasures，算法实现、清L1 cache

## 相关技术研究

![2020-0721-SoK%20Vuls%20in%20TEE%20773876bae5ff46ccba6b47cda476d3cf/Untitled%2011.png](/images/2020-09-08/Untitled%2011.png)