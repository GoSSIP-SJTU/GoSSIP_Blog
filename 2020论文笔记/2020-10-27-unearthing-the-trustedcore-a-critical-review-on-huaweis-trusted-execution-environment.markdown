---
layout: post
title: "Unearthing the TrustedCore: A Critical Review on Huawei's Trusted Execution Environment"
date: 2020-10-27 15:35:19 +0800
comments: true
categories: 
---

> 作者：Marcel Busch, Johannes Westphal, Tilo Müller
> 
> 单位：Friedrich-Alexander-University Erlangen-Nürnberg, Germany
> 
> 会议：29th USENIX Security Symposium Woot 2020
> 
> 原文：https://www.usenix.org/system/files/woot20-paper-busch.pdf

## Abstract

TEE 是现代移动设备安全的核心，在本文中，作者针对华为的 TEE 实现：TrustCore 进行了研究，并发现了多个软件栈中的设计缺陷，影响 2016 年发布的 Huawei P9 Lite 到 2018 年发布的 Huawei P20 Lite。

1. 作者对 TrustCore (TC) 的组件、交互与 Android 系统之间的整合针对安全方面进行了逆向工程
2. 作者检查了 TC 的 TA loader 并发现了多个设计问题。这些问题会允许攻击者解密任何目标设备上的 TA，并破坏其代码保密性。
3. 作者描述了华为的 Keystore 系统设计，并在 Keystore 系统中发现了许多严重的漏洞，并示范了从被 TEE 保护中的 Keysotre 导出明文密钥
4. 在以上这些发现中，作者还在华为 Keymaster TA 中发现了内存破坏漏洞，可以在 ARM TrustZone 最高执行权限层执行恶意代码。同时，栈 Canary 和 ASLR 的设计缺陷可以使攻击者绕过这些保护

作者向华为报告的本文中的发现，遵循了华为的 responsible disclosure 流程，并第一次在本文进行公开讨论作者的分析

<!-- more -->

## Introduction

TEE 是现代移动设备安全架构中不可或缺的一部分。他们提供了需要最高安全的服务的执行上下文，例如用户鉴权、移动支付与数字权利管理，并在 Rich OS 之外隔离执行。Rich OS 也就是 Android 或 iOS，拥有复杂的软件栈，容易产生错误。理论上，在 Rich OS 中的任何级别的漏洞包括内核漏洞，都不会影响到 TEE，因为 TEE 是使用硬件来进行隔离的。ARM 架构中使用 ARM TrustZone 来提供这种硬件隔离，包括本文讨论的华为设备

尽管 TEE 已经在数百万设备上使用了数年，针对这些系统的安全性分析却很少，包括华为、高通和三星。但是，正确的 TEE 实现相对于实现操作系统来说要复杂且困难的多，这些挑战通常会引起潜在且非常严重的安全问题，影响到整个设备的安全架构

在本文中，作者主要关注华为的 TEE 实现：Trust Core，被应用在 2016 年发布的 Huawei P9 Lite 到 2018 年发布的 Huawei P20 Lite 中。在更新的设备上例如 Huawei P40 和 Huawei P30，TEE 架构和实现已经改成了 iTrustee

1. 本文 review 是首个对整体架构描述进行的，作者结合了动态和静态分析，逆向分析了 TC 的组件并系统描述了其与 Android 系统的整合
2. 其次，作者展示了多个设计缺陷导致的保密性 TA 可以被直接解密得到明文二进制文件。作者通过 TA loader 加载加密的 TA 进 TC 的过程中分析了 TA loader 的这种解密操作。作者在多个华为 P 系列设备的 TEE 中发现了 TA loader 的解密密钥完全一致
3. 然后，作者分析了华为实现的 Android Keysotre 系统，并描述了这种实现中的多个极其严重的设计问题，并演示了从硬件保护的系统中导出明文密钥
4. 最后，作者发现了可以在最高执行权限的 TEE 上下文中执行恶意代码的漏洞，并绕过了操作系统的 Stack Canary 与 ASLR 保护

## Background

### ARMv8-A 权限层次

![1](/images/2020-10-27/uploads/upload_8be16919f308971a82a349ea7d87a856.png)

1. 独立内核，独立页表
2. NW 无法访问 SW 的内存区域，SW 则没有任何限制
3. 最高层为 Secure Monitor

在所有的可用商业平台，一个安全引导链仅允许厂商签名的软件在 SW 中执行，因此所有的 SW 组件都是系统的 TCB (Trust Computing Base) 的一部分。与之相对的，NW 中则允许开发者在解锁 Bootloader 后对 Rich OS 的内核进行修改（但是华为在 2018 年 5 月之后并不提供解锁 Bootloader）

SW 还被 SoC 系统设计者设计允许访问外设，例如指纹传感器。在这种使用场景下，SW 提供 API 给 NW 去调用，而永远不允许 NW 访问指纹传感器或其指纹原始图像

### Android 中的 TEEs

由于厂商不同，TEE 的实现也不尽相同

* **高通**: Qualcomm Secure Execution Environment (QSEE)
* **三星**: Exynos: Kinibi; Trustonic; TEE-Gris
* **华为**: TrustedCore (TC) 

Android 系统使用的 TEE 可以增强很多服务，包括用户鉴权，全磁盘加密等。例如，所有的 Android 系统实现了用户验证使用三个以 TEE 为后端的 NW 组件：

1. **Gatekeeper**: 用户身份验证（PIN、指纹）
2. **Fingerprintd**: 基于指纹的用户认证
3. **Keystore**: 给以上两个服务提供密码学基础的服务

以 TEE 为后端的全盘加密由卷守护进程 (`vold`) 与其交互的 TEE 中的 Keymaster TA。密码学密钥仅仅在 TEE 中明文存在，从而保证了所有加密的分区只能在其设备上解密。这种设备绑定性杜绝了了暴力破解用户 PIN、密码或验证图案

### 攻击模型

作者提出的攻击模型基于所有的 NW 组件都是不可信的。该攻击模型适用于所有基于 TZ 的 TEE 系统，更现实地说即基于 Android 的移动设备。在本文中，所有演示的攻击都基于攻击者可以对 N-EL0 的完整控制权，即 root 级别攻击者

值得注意的是，沙箱化的 Android 应用通常不足以执行作者提出的攻击，但是 TEE 设计之初就是为了在极高权限的攻击威胁下仍然能保证安全。因此在本文中提出的攻击模型中，拥有 root 或内核权限是合理的

但是，作者无需修改并在 N-EL1 中修改并执行代码。在权限提升攻击中，因为系统服务访问特定内核模块的权限是充足的，作者甚至不需要 N-EL0 的 root 执行权限

## TrustCore 架构逆向

![2](/images/2020-10-27/uploads/upload_e46f64f93debd8f1cedb9e4f732f691e.png)

### Normal World

核心使用 TEE feature 的组件即 Android 系统服务。这些系统服务使用 Binder IPC 框架暴露他们的功能给 App 或其他系统服务。因为不同的 SoC 的 TEE 通常不一样，因此 Android 定义了一层硬件抽象层，即 HAL。这个 HAL 库实现系统的 HAL 服务接口

在华为的 TEE 实现中，主要是 libteec。HAL 库主要负责两个任务：1. 将传给通用 HAL 接口的数据结构转换为 TEE 实现的数据结构；2. 描述 TEE 的状态机。

一个通常的使用 GlobalPlatform (GP) TEE 客户端 API 的序列在华为设备上一般以以下顺序

1. **`TEEC_InitializeContext`**: 客户端连接到 `teecd` 开放的 UNIX socket `/dev/tc_ns_client`，然后 `teecd` 传递与文件描述符相关联的登录凭证到 TEE 驱动中
2. **`TEEC_OpenSession`**: 在有初始化好的上下文后，客户端可以打开一个 session 连接 TA。首先它会检查 TA 是否已经加载，否则尝试加载其 TA。在 SW 中，`globaltask` 扮演着 NW 中类似 `init` 的角色，负责加载 TA
3. **`TEEC_InvokeCommand`**: 客户端在 TA 执行命令
4. **`TEEC_CloseSession`** 与 **`TEEC_FinalizeContext`**: 客户端结束 session

### Secure World

NW 的组件可以静态与动态分析，但是 SW 组件的研究的观察则被限制在与 NW 交互上

SW 的组件主要包括以下几部分

* Trust OS 内核: `TrustedCore`
* `init` 进程: `globaltask`
* 普通进程: 各种 TA 与 Keymaster TA

1. **`TA_CreateEntryPoint`**: 在 TA 的生命周期中只会调用一次
2. **`TA_OpenSessionEntryPoint`**: NW 客户端 `TEEC_OpenSession` 直接相对应的操作
3. **`TA_InvokeCommandEntryPoint`**: NW 客户端 `TEEC_InvokeCommand` 直接相对应的操作
4. **`TA_CloseSessionEntryPoint`**: 释放所有 session 特定的状态
5. **`TA_DestroyEntryPoint`**: 释放所有 TA 分配的资源

## Breaking Code Confidentiality of TAs

### 加载加密的 TA

![3](/images/2020-10-27/uploads/upload_8c2217a34fcce2563ecc1531ef3a404a.png)

### 提取 TA 解密密钥

以上这种加载机制，引出了一个非常重要的问题：RSA 私钥是如何被保护的。理论上来说，RSA 私钥永远都不应当被加载至内存中，所有的操作都应当使用 TPM 来进行操作。只有这样，即使 `globaltask` 被攻击，RSA 私钥也不会被泄漏。然而很遗憾的是，`globaltask` 使用了白盒加密算法来保护 RSA 私钥，这使得攻击者可以直接执行这种白盒算法来解密 RSA 私钥。

作者使用了如下实例代码来解密密钥。

![4](/images/2020-10-27/uploads/upload_aaf7e1f4a7639b88a8dc69d7313a543a.png)

![5](/images/2020-10-27/uploads/upload_37bb157886b3e3ee08d204ffd50d9aec.png)

## Extracting Export-Protected Keys

### Android Keysotre 系统

Keysotre 使用 Android binder IPC 框架与 TEE 中的 Keymaster TA 进行通讯，完成密码学操作

### 提取 Keymaster 的 Master Keys

![6](/images/2020-10-27/uploads/upload_05dc172eb6522f7c504246710ee62255.png)

keymaster TA 使用 HMAC 算法来检查 keyblob 是否合法。HMAC 使用的密钥是用 AES-CBC 加密的，但是用来加密密钥的 AES key（KEK）是一个常量，使得作者可以在 TEE 外解密 HMAC 的密钥。

除了 HMAC，HMAC-sha512 也使用了一个与 KEK 不同的常量 key。因此这两个 key 都在 TA 的虚拟内存空间中，且在 TA 的二进制文件中，并非从 TrustOS 或特殊硬件中动态加载。

### 攻破全盘加密

Android 的全盘加密的安全性保证基于 TEE。这种硬件层次上的安全保证可以保证全盘加密不会被离线通过暴力破解的手段破解用户的 PIN、解锁图案或密码。

由于上一节中讨论的问题，这种全盘加密可以通过解密 Device Encryption Key 来解密用户数据分区。与高通方案不同的是，高通使用了每个设备均不同的密钥，而华为的所有设备都使用了完全一致的密钥。因此，攻击者可以发起离线暴力破解设备的全盘加密。

## Writing a Keymaster Exploit

### 任意代码执行

作者在 RSA 公钥导出操作中找到了基于栈的缓冲区溢出漏洞，从而可以破坏内存，并劫持 Keymaster TA 的控制流

虽然有 Stack canary 与 ASLR 的保护，但是 Stack canary 是一个常量，且可以被覆盖为其他已知的值；ASLR 则缺乏暴力破解的保护，因为在 TA 崩溃之后地址空间与加载之前一致

### 权限提升

一个典型的有用的 sysycall 可以让 TA 映射物理内存地址到他们的虚拟地址空间。这种函数的实现接受一个物理地址，一个大小与一个 flag。这个 flag 又被称之为安全模式 flag

因为这个 syscall 直接暴露给了 TA，攻击者可以关闭安全模式 flag 从而绕过内核的检查，并映射任意物理内存地址到其虚拟地址空间，包括更高权限执行层的物理地址，例如 S-EL1 或 S-EL3

## Lessons Learned

1. 硬件保护密码学密钥的缺乏
2. TCB 攻击平面

## Related

[Trusty TEE](https://source.android.com/security/trusty)
