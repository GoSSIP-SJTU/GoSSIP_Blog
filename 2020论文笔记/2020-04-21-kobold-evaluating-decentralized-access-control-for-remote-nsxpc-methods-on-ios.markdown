---
layout: post
title: "Kobold: Evaluating Decentralized Access Control for Remote NSXPC Methods on iOS"
date: 2020-04-21 14:26:12 +0800
comments: true
categories: 
---

> 作者：Luke Deshotels, Costin Carabas, Jordan Beichler, Ra ̆zvan Deaconescu and William Enck
> 
> 单位：North Carolina State University, University POLITEHNICA of Bucharest
> 
> 出处：2020 IEEE Symposium on Security and Privacy (S&P 20)
> 
> 原文：[Kobold: Evaluating Decentralized Access Control for Remote NSXPC Methods on iOS](https://www.computer.org/csdl/pds/api/csdl/proceedings/download-article/349700a399/pdf?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJjc2RsX2FwaSIsImF1ZCI6ImNzZGxfYXBpX2Rvd25sb2FkX3Rva2VuIiwic3ViIjoiYW5vbnltb3VzQGNvbXB1dGVyLm9yZyIsImVtYWlsIjoiYW5vbnltb3VzQGNvbXB1dGVyLm9yZyIsImV4cCI6MTU4NTYzODg5OH0.mM9nx6cZD1ek9t20eGEB5TxoGyTiQ8e-ZU4i2e7bmyE)

## Abstract

苹果使用沙箱、文件访问控制等访问控制机制来防止第三方应用程序直接访问安全敏感的资源。但是第三方应用程序可能通过与系统守护进程（system daemons）进行进程间通信（IPC）来间接访问这些资源。如果这些守护进程无法在IPC上正确地执行访问控制，则可能导致混淆代理漏洞(confused deputy vulnerability)。

在本文中，作者提出了Kobold框架，结合静态和动态分析来研究基于NSXPC的系统服务。使用Kobold，作者发现了多个具有混淆代理漏洞和守护进程崩溃的NSXPC服务，另外还包括激活麦克风、禁止访问所有网站以及泄露存储在iOS文件提供程序中的私人数据。

<!-- more -->

<!-- > 作者近几年的论文：
> * CCS'2016: SandScout: Automatic Detection of Flaws in iOS Sandbox Profiles. In Proceedings of the ACM Conference on Computer and Communications Security (CCS). Vienna, Austria.
> * AsiaCCS'2018: Automated Evaluation of Access Control Policies in iOS. In Proceedings of the 2018 ACM on Asia Conference on Computer and Communications Security (AsiaCCS). Incheon, Korea. -->

## 1 Introduction & Background

苹果为了保护用户，第三方应用程序在沙箱范围内运行，限制了直接访问资源的数量。但是应用程序还可以通过与系统守护进程的进程间通信（IPC）间接访问敏感资源。例如，应用程序无权直接访问用户的日历，但可以使用日历管理守护进程提供的服务来查看和修改日历事件。如果守护进程未正确执行访问控制，则第三方应用程序可能会滥用该进程作为混淆代理来执行某些损害系统或侵犯用户隐私的操作。

### 1.1 Mach IPC 及其访问控制

iOS的内核XNU结合了Mach微内核、FreeBSD和驱动程序框架（I/O Kit）：
* Mach通过Mach消息提供进程间通信（IPC）功能。 
* FreeBSD提供了文件系统和TrustedBSD强制访问控制（MAC）框架，该框架允许苹果hook系统调用并实现沙箱。
* I/O Kit 作为用户空间和内核空间之间的接口，通常是模糊测试的目标。

IPC基于Mach微内核，其原始组件是**mach消息和mach端口**，提供服务的进程可以通过注册mach端口的名称来提供远程方法，客户端可以向该端口发送消息以调用远程方法。如果消息格式正确且客户端具有足够的权限，则服务器将执行客户端调用的方法<!-- launchd low level, Kobold 不研究这点 -->

**XPC和NSXPC**：编码和解码mach-message的过程很复杂，因此苹果提供了相关的抽象以使IPC对开发人员更简单。最新的接口类型是XPC及其面向对象的版本NSXPC。在面向对象的IPC中，对象及其方法驻留在提供服务的进程中，但是客户端可以访问该对象，就好像它存在于客户端的地址空间中一样。因此，使用NSXPC的服务提供进程可以注册多个mach-port名称，每个名称都提供对远程对象的访问，并且每个远程对象都公开远程方法。 

**NSSecureCoding**：为了减轻类型混淆攻击，使用NSXPC公开的远程方法具有严格的参数类型，必须遵守协议NSSecureCoding。因此，研究过程中必须完成三点：1.识别与NSXPC接口关联的mach端口；2. 查找远程对象提供的远程方法的名称；3. 获得这些远程方法的预期参数类型。

**Entitlements**是与IPC访问控制最相关的功能，它是静态嵌入到可执行文件代码签名中的键值对，且只能在官方的应用程序更新中进行更改（安装该应用程序的用户不可见）。 Entitlement用来确定应用程序可以访问哪些特权：
* 危险的entitlement（例如，绕过代码签名限制）是私有的，保留给苹果创建的可执行文件
* 不太敏感的和公开的entitlement可以由第三方开发人员使用
  > iCloud, push notifications, Apple Pay
* semi-private entitlement无法由Xcode中获取，但在App Store上的许多第三方应用程序中可以找到。

![](/images/2020-04-21/NSXPC_access_ctrl.png)

**Enforcement**有三个阶段：
* 第一阶段，沙箱可以允许或拒绝连接到特定mach端口名称的请求（例如会导致launchd进行连接的系统调用)。 但苹果必须允许第三方应用程序访问某些IPC功能（例如访问位置数据），因此它不能使用沙箱来阻止对所有mach端口的访问。
* 第二阶段，服务提供进程可以根据客户端的capability来接受或拒绝尝试连接到mach端口的尝试。
* 第三阶段，每个远程方法都可以根据客户端的capability来接受或拒绝调用它们的尝试。

对于第二、三阶段，服务提供过程可以使用`SecTaskCopyValueForEntitlement`API检查客户端的entitlement。该API允许进程指定entitlement key和客户端（即代表客户端ID的token），并且API将为客户端返回与该entitlement密钥相关联的值。

### 1.2 研究问题与挑战

**现有的相关安全工作：**
* 基于IPC的混淆代理漏洞之前也有一些工作：Woodpecker使用预加载的Android应用程序上的数据流分析来枚举暴露给其他应用程序的危险服务。但是例如方法调用的动态分配等objective C的特点使数据流分析难以在iOS二进制文件上进行。
  > 实际上也可以，只不过在效率和粒度上需要有所取舍，另外可能需要一些预处理
* 目前没有系统枚举可供第三方应用程序访问的iOS远程方法的工作，与之最接近的相关工作是针对iOS的现有IPC fuzzer，用于探测代码缺陷（例如类型混淆或取消引用漏洞），这些缺陷可被利用来获取任意代码执行。但是这些fuzzer不会尝试枚举远程方法或识别混淆代理漏洞
* 从策略的角度来看，作者之前的工作`iOracle: Automated Evaluation of Access Control Policies in iOS`和`SandScout: Automatic Detection of Flaws in iOS Sandbox Profiles`系统研究了基于文件的访问控制。

具体来说，作者研究的问题是：**第三方应用程序可以访问哪些对安全性和隐私敏感的NSXPC方法**？
<!-- 其答案代表了远程方法的攻击面，第三方方法可能会通过IPC利用这些方法。 -->

为了回答这个问题，作者解决了三个研究挑战：

* **首先，可用于第三方应用程序的entitlement集是未知的**。作者识别了第三方应用程序可以使用的两套entitlement：
  * 所有开发人员均可获得的公开集合
  * Apple仅向选定的开发人员提供的半私有（semi-private）集合。
* **其次，第三方应用程序可访问的NSXPC服务集是未知的**。提供这些服务的可执行文件是闭源，并且没有集中的策略来对这些服务和entitlement进行映射。具体来说，NSXPC服务通过服务名称动态解析，将NSXPC服务（即方法名称）映射到相应daemons和daemons中的IPC入口点均没有文档或配置文件说明。此外，用于访问NSXPC服务的访问控制策略被硬编码到daemons中。因此无法查阅以专有格式或其他格式编码的策略规范。 也无法自动确定entitlement检查的位置（例如，它们可以间接在库中进行），也无法确定哪些资源受到entitlement检查的保护。
* **第三，不知道哪些NSXPC服务在安全性或隐私方面是敏感**。这些服务的语义没有公开文档，且在iOS中的数据流分析也非常困难。

在本文中，作者提出了Kobold框架，用于研究iOS中的NSXPC服务。 Kobold基于了两个关键的观察：
* 首先，标准化的IPC接口（例如NSXPC）在编译后的代码中包含可预测的模式，这些模式可通过静态分析进行识别。
* 其次，未经授权而访问IPC服务的尝试会返回错误消息，可以提供iOS IPC访问控制策略的模型。

利用这些观察，Kobold提供了基于模式的静态二进制程序分析来枚举NSXPC接口，然后动态地使用系统探测来提取大致的访问控制策略（由给定服务中的条件检查编码）。作者使用Kobold研究了iOS 9、10和11，并发现了多个NSXPC服务，这些服务具有混淆代理漏洞和守护程序崩溃。发现的漏洞使第三方应用程序可以激活麦克风、禁用对所有网站的访问以及泄漏存储在iOS File Provider中的私人数据。所有问题均已报告给Apple并有两个CVE。

作者主要作出了三点贡献：
* 提出了Kobold，这是第一个评估iOS系统代码中实现的NSXPC访问控制策略的框架。 Kobold枚举了第三方应用程序可访问的NSXPC服务，并启发式地去判断哪些服务是可被利用的。
* 对semi-private entitlement进行了首次评估。作者分析了大约六千个受欢迎的第三方应用程序和十万个随机选择的第三方应用程序，以确定哪些entitlement是苹果分配给未公开的第三方开发人员的。
* 确定了以前未知的安全问题，其中包括三类混淆代理漏洞和十四个守护程序崩溃。作者的发现包括root权限守护程序崩溃，对移动设备管理（MDM）功能的非特权访问以及未经用户许可的麦克风激活。

<!-- Kobold不需要越狱的设备，但越狱设备可以提供有助于识别漏洞的额外数据。此外，Kobold还可以用于研究发行的新版本，具有迁移性。 -->

## 2 Overvier

作者通过静态分析和动态测试来解决上述挑战。

* 首先对服务进行枚举（即，标识端口，方法名称和参数）
* 其次，对第三方应用程序可访问的那些服务进行分类
* 第三，使用试探法来选择可访问的，对安全敏感的服务以进行手动分析。

![](/images/2020-04-21/overview.png)

<!-- Kobold的静态分析有助于枚举攻击面，而动态分析则使分析人员可以对哪些NSXPC服务可能包含漏洞进行分类。仅使用动态分析的简单方法可能会忽略一些在运行时很少调用的服务。同样，仅关注静态分析的方法可能会花费大量时间分析第三方应用程序实际上无法访问的服务。 -->

**服务枚举**：查找IPC漏洞的常用方法是在正常系统活动期间动态记录IPC消息，并以细微的mutation来重播这些消息。但是，这种“记录和重放”方法有两个缺点：
* 首先，它不会显示在“记录”阶段未调用的很少使用的服务。
* 其次，它高度依赖使用越狱设备来记录IPC活动。

作者则采用静态分析从iOS固件中提取面向NSXPC服务（即NSXPC）

**对可访问服务进行分类**：通过对第三方应用程序可访问的服务进行分类，可以节省大量的人工。作者使用三种技术来执行此分类：
* 首先，使用iOS沙箱策略模型来确定第三方应用程序可以访问哪些端口。
* 其次，使用一个iOS应用程序动态调用服务。大量服务以completion handlers (callbacks)的形式提供响应，这些响应使作者可以确认何时成功访问了服务。
* 第三，沙箱模型和服务响应（例如，错误消息）有时表明访问服务需要特定的entitlement。为了确定第三方应用程序是否可以拥有所需的entitlement，作者对Apple App Store进行了自动化调查，并创建了一个entitlement的列表。

**漏洞分析**：初始动态测试使用未初始化的值作为变量传递给服务的参数。在许多情况下，未初始化的值足以触发异常的系统活动（例如crash、向用户提示、禁用系统资源、声音警报）。执行其他测试从而将特定的服务调用与观察到的安全敏感操作之间进行映射。作者还将方法名称用作启发式方法，以优先考虑人工调查的方法。在人工调查期间，作者可以使用有效值初始化变量，并且可以选择使用越狱设备来监视系统活动（例如，文件访问日志），同时调用服务。

## 3 Kobold

Kobol主要分为三个任务：
* 首先，对第三方应用程序可用的entitlement进行调查。 
* 其次，枚举第三方应用程序可访问的NSXPC服务。 
* 最后，评估可访问的NSXPC服务的安全敏感性，以找出可能遭受混淆代理攻击的服务。

第一项和第二项任务是自动化的，第三个任务使用模糊测试和手动分析来研究可访问且安全敏感的NSXPC服务方法。

### 3.1 Identify Semi-Private Entitlements

由于沙箱中的entitlement要求可以确定第三方应用程序可访问的mach端口集，因此第一步是枚举第三方应用程序可以拥有的entitlement。作者为iOS应用启用Xcode中所有可用的功能，并从该应用的签名中提取entitlement。但是识别semi-private entitlement需要从App Store抓取应用程序并调查其权利。

<!-- 定义（半私人权利）。如果某个权利由应用商店上的第三方应用拥有，但启用所有功能的实验性Xcode应用所不拥有，则该权利为半私有。 -->

Kobold的entitlement调查框架分为两个阶段。首先，从Apple App Store自动下载代表iOS应用程序的.ipa文件。其次，从每个.ipa文件中提取元数据和权利数据，并搜索尚未标记为public的entitlement，此处需要忽略将Apple列为开发人员的App Store应用程序中的entitlement。此类应用程序默认情况下未安装在iPhone上，但由于其代码库归Apple所有，因此仍可以被授予private权利。

**App Scraper**:作者为Kobold开发了一个app爬取工具，使用可访问性选项和AppleScript在macOS上模拟iTunes，并于2017年9月收集了后续实验的应用程序。
<!-- Apple已正式从默认版本的iTunes中删除iOS应用程序市场。 但是为了重现该分析，可以安装iTunes的另一个版本，并恢复iOS应用程序市场的功能。 -->

**Modeling Sandbox Entitlement Checks**:Kobold通过两种方式扩展了现有的iOS访问控制模型(iOracle)：
* 为mach端口访问建模沙箱规则。
* 列举可用于第三方应用程序的权利。 

通过结合这两种方式，Kobold可以自动将第三方获取的entitlement映射到需要这些entitlement的沙箱规则。 通过此映射可以推断哪些mach端口是沙箱化的第三方应用程序可以访问的（即使该应用程序具有semi-private权利）。

### 3.2 Enumerate Accessible NSXPC Services

要调用NSXPC服务，客户端必须正确指定 **mach端口名称** 和 **与该服务关联的远程方法**。Kobold使用两种静态分析技术和一种动态分析技术来找到这些mach端口和方法，以便枚举第三方应用程序可访问的NSXPC服务。

<!-- 首先，从从iOS固件提取的配置文件缓存中提取mach端口与托管它们的可执行文件的映射。其次，从守护进程二进制文件中静态提取包含NSXPC服务方法名称的协议标头。第三，作者开发的应用程序尝试在记录这些调用的响应的同时调用mach端口和方法名称的组合。 -->

**Mapping Mach Ports to Executables:** 通过分析iOS固件中提取得到的xpcd_cache.dylib中存储的mach-port名称注册的缓存，可以静态获得mach-port到可执行文件的映射。首先，Kobold在`.dylib`二进制格式标识代表.plist文件的section，并使用jtool提取该部分。然后，使用plutil将该plist文件从二进制格式转换为xml。最后使用正则表达式解析xml格式的plist文件，以提取从提供服务的可执行文件到给它们所托管的mach-port的映射。

> 通常，iOS将Mach端口名称静态地映射到承载它们的可执行文件。采用静态而非运行时设置服务是因为：当请求该daemon提供的服务时，静态映射允许launchd启动相应的守护程序。静态映射还可以防止进程伪装为主机端口以窃取IPC消息。

**Mapping Protocols to Executables:**

由于NSXPC是面向对象的接口，因此客户端和服务提供者都应具有称为协议（protocol）的方法声明（方法名称和参数类型）的列表。但这些协议不是公开可用的，必须从iOS固件映像上找到的服务提供者的可执行文件进行提取。

作者使用静态分析工具class-dump从iOS守护程序的可执行文件中提取面向对象的协议方法声明，进而搜索提供与NSXPCConnection或NSXPCListener相关联的接口类的提供服务的可执行文件。这里提取的协议不能保证代表NSXPC服务，但是作者将它们视为可以通过动态测试进行进一步筛选的大致集合。例如，`NSXPCListenerDelegate`协议经常出现，但它充当支持其他NSXPC服务连接的实用程序服务，与本文分析无关。

<!-- 此时，mach端口已映射到可执行文件，而协议已映射到可执行文件。另外Kobold还删除了沙箱阻止访问的mach-port -->

**Mapping Ports to Protocols:**

由于一个可执行文件可以使用多个mach端口和多个protocol。因此尽管当前已经大大减少了可能的组合，但仍然需要消除无效的mach-port与可执行文件中的协议组合的歧义。 Kobold通过在运行时尝试每种组合并使用消息反馈来确定哪个mach端口到协议的组合是有效的，从而解决了这种歧义。

**Bypassing Compile Time Policies:**

Xcode IDE禁止开发人员在其iOS代码中调用NSXPC API。但是通过逆向，作者确认iOS上的系统程序确实使用NSXPC，也就是说，用于NSXPC的库存在于iOS设备上，但是Xcode在编译时会阻止这种调用以防止恶意或滥用底层功能。为了明确起见，第三方应用程序应调用 从第三方应用程序的地址空间间接调用NSXPC API的 库。如果开发人员直接调用NSXPC API，则他们更有可能使用无效或危险的参数来调用它们。作者对iOS SDK中的NSXPC头文件进行调查后发现，所需的NSXPC API带有`__IOS_PROHIBITED`标签。从头文件中删除这些标签就能使用Xcode来编译带有NSXPC API的应用程序。

**Completion Responses:**

一旦客户端连接到mach端口，NSXPC便允许它调用关联的远程方法。许多方法都包含一个称为completion handler的特殊参数，该参数包含零个或多个参数以及远程方法完成后要执行的代码块。Kobold调用与提取的协议相关联的所有远程方法，并假定触发completion handler响应的消息是可访问的，除非返回要求特定entitlement的错误消息。这些完返回的错误可能可以指定调用方法所需的entitlement的键值。

<!-- 自动生成的完成处理程序示例如图3所示。 -->
<!-- ![](./image_kobold/ex_completion_handler.png) -->

### 3.3 Security Sensitivity of NSXPC Services

作者使用四种方法评估每个远程方法的安全敏感性并筛选潜在攻击：
* 使用方法名称的语义和entitlement要求的不一致性进行方法的分类方法以做进一步的人工调查；
* 手动调查通过completion handler参数返回的值；
* 观察用户在设备上可感知的变化；
* 使用越狱设备来提供对文件操作和崩溃日志的补充了解。

**Method Name Semantics:** 苹果尚未混淆远程方法的名称且Objective-C要求在方法名称中提及每个参数，因此方法名称包含大量与功能相关的语义信息。具有成功的completion handler的每个方法的名称均进行手动审核。例如，由于“Recording”，“Dictation”和“Speech”，作者将以下方法声明手动分类为对安全敏感的。

![](/images/2020-04-21/ex_r_d_s.png)

**Entitlement Inconsistencies:** 作者提供两个假设：
* 与mach端口关联的每个方法都具有相似级别的安全敏感性；
* 要求授权的方法对安全性敏感。

因此，任何不需要授权且与需授权的方法共享端口的方法都被认为是安全敏感的。

**Observations Without Jailbreak:** 在未越狱的设备上对远程方法fuzzing时，可以观察到大量的系统活动。许多方法都包含代表返回值的参数，并且这些方法执行完后，这些值可能包含对安全性敏感的数据。
* 第一步，作者方法参数初始化为简单的值，例如数字0或空字符串。
* 然后，作者使用先前通过静态分析或动态分析收集的值，例如打开文件的名称或程序中使用的文件名和字符串。如果无效的方法参数导致设备功能被破坏（例如互联网访问、配置选项），则可以通过手动调查设备状态来检测到这些更改，另外fuzzing过程中应用程序发生的声音或提示之类的影响也可以记录下来。
* 最后，通过“设置”菜单，iOS用户可以看到崩溃报告。这些报告可用于检测由调用越狱设备上的方法而引起的crash。

## 4 IDENTIFIED SEMI-PRIVATE ENTITLEMENTS

应用程序的entitlement在确定可访问的端口和远程方法方面起着重要作用。public entitlement可以通过Xcode轻松实现，但是苹果还向一部分第三方开发人员分发了一组未知的semi-private entitlement。 因此需要找到**第三方应用程序可以获取哪些semi-private权利**。 作者于2017年9月和10月对Apple iOS App Store进行了调查。

**Six Thousand Popular Apps:** 有25种应用程序类型，每种列出了美国排名前240位的最受欢迎app，总共6000个app。除去类别之间交叉重复的共5873个，其中免费的有5716个但16个在美国不可用。于是作者的最终样本集包括当时在美国提供的5700种流行的免费应用程序。 由于其中17个app将Apple列为开发人员，因此作者将其余5683个标记为第三方app。

**100k Random App Sample:** 作者还收集了10万个随机选择的应用程序。作者开发了工具自动执行，花了两周时间下载、花了两天时间来提取这entitlement。在此样本中并未发现新的semi-private entitlement。因此作者认为之前6000个的数据集合已经足够了，后面都是基于该数据集实验。

**Results:** 作者发现了17个semi-private entitlement。据作者所知，只有4个有苹果提供的申请过程文档：pass-presentation-suppression，payment-pass-provisioning，previous-application-identifiers和HotspotHelper。

![](/images/2020-04-21/table_1.png)

**Vendor Specific Entitlements:** 半私有权利中的五种是特定于供应商的，在权利密钥中列出了应用程序开发人员的名字

<!-- Check it -->
**Sharing Resources with Daemons:** 两个应用程序具有唯一的半私有权利，这些特权似乎可以被系统应用程序使用。例如之前披露的Uber应用程序具有explicit-graphics-priority的权利，越狱应用程序使用该权利来构建录屏应用程序。这种相关性意味着Uber可以在后台运行时记录用户的屏幕。此外，Netflix还具有allow-mpeg4streaming的权利，内置于iOS的系统应用程序也拥有此权利，但该权利尚未记录。

## 5 Empirical Study of NSXPC Attack Surface

除了搜索混淆代理攻击外，作者还使用Kobold对NSXPC方法进行定量分析。此分析枚举了可访问的方法并度量了各种特性，例如每种方法所需的参数数量和类型。还研究了NSXPC方法的权利要求。

![](/images/2020-04-21/fig4.png)

**Hierarchical Results:** 分层结果：图4说明了使用Kobold在iOS 11.3.2设备上使用仅具有默认权利的应用程序动态测试的`the number of invocations, unique methods, completion handlers, completion confirmations, and entitlement free methods`结果。

Kobold的静态分析分析了提取的276个沙箱可访问的mach端口和3048个候选远程方法来调用:
* 使用与每种方法提取自的守护进程相关的mach端口测试了1517种独特的方法。由于mach-port到协议的映射不明确，可能会将许多方法分配给错误的端口。
* 测试的方法中有677个包含completion handlers，而这些方法中的224个在调用时返回了completion handler confirmations。
* 在具有成功完成消息的224个远程方法中，有139个不需要授权，有8个不需要特定entitlement，有77个需要特定entitlement。

**Completion Handlers:** 其参数（例如NSError或NSString值）可以使用守护程序中的数据来初始化，并在其代码块范围内使用。 Kobold使用此代码块输出completion confirmation，此输出能够确定带有完成处理程序的远程方法是否已运行。但是大约一半unique method没有完成处理程序，未经进一步分析就不能标记为可访问或不可访问。

**Impact of Entitlements on NSXPC Services:** 作者确定了两个有条件的沙箱规则来允许基于public或semi-private entitlement访问mach端口。结果由表III表示。错误消息中指定权利要求的方法仅指定了私人权利（App Store上的第三方应用程序无法访问的私有权利）。与半私有权利关联的端口未映射到任何NSXPC方法（也许它使用了另一种IPC接口类型）。与Siri权利相关的端口映射到221个潜在的远程方法调用。但是，即使调用应用程序具有Siri权利，动态测试也无法使任何这些方法调用触发完成处理程序响应。

![](/images/2020-04-21/table3.png)

**Number of Arguments:** 方法声明中的参数数量在确定成功调用远程方法的难度以及是否可以利用该方法方面起着重要作用。例如，参数为零的方法正确调用很简单，但是不太可能被利用，相反，包含10个参数的方法具有较大的攻击面，但是可能很难找到所有参数的有效值。表IV列出了具有各种参数数量（即0到10个参数）的方法的数量。对于远程方法，完成处理程序被视为单个参数，但是完成处理程序可能具有自己的参数。
<!-- 请注意，这些值是使用所有1517个提取的潜在NSXPC方法推论得出的。 因此，可能会包含无法访问或误报的方法（即未远程公开的方法）。 -->

![](/images/2020-04-21/table4.png)

**Types of Arguments:** 表V列出了Kobold调用的方法的声明中最常出现的数据类型。作者将这些数据类型分为三类，即primitives，documented和undocumented。基本类型由以C语言显示的那些low level类型组成（例如，int，long，double）。documented类型（例如，NSString）是在原始类型上构造的抽象，Apple正式对其进行了文档记录。 undocumented（例如AFSpeechRequestOptions）是基于原始类型的抽象，但是Apple尚未正式记录这些数据类型。尽管可以使用随机值来fuzz原始值，但是很难找到更复杂类型的可接受值。Apple的文档可能会提供有关已记录类型的初始化的提示，但是要对NSXPC远程方法所期望的值进行全面分析，可能需要对远程方法进行动态分析，符号分析或大规模逆向。

![](/images/2020-04-21/table5.png)

**Intra-Port Entitlement Consistency:** 表VI列出了具有成功完成处理程序及其各自的mach端口的方法的数量。 这些方法也分为两类：1）不需要权利的方法； 和2）确实需要授权的人。 从完成处理程序中提供的错误消息中推断出权利要求。 在两种类别中使用方法的mach端口被认为具有不一致的权利策略，并在表VI中进行了突出显示。 如第VII节中的MDM功能所示，不一致的授权策略可能表示访问控制漏洞，在这种情况下，安全特权方法会意外地提供给非特权客户。

![](/images/2020-04-21/table6.png)

**Triage strategy:** 在图4所示的全部3048次调用中，在几次测试活动中调用了1517种独特的方法，同时监视了手机的未定义或意外行为。触发此类事件后，便会启动带有方法子集的新测试活动。

## 6 Findings

Kobold发现了混淆代理漏洞和守护程序崩溃，这些发现在iOS 11越狱设备上被揭示，并在iOS 12设备（在文章撰写时为最新的主要版本）上复现。作者以PoC的形式向Apple披露了本文发现。

### 6.1 Confused Deputy Vulnerabilities

表VII列出了Kobold发现的混淆代理漏洞。这些漏洞可以分为三类：1）文件提供者信息泄漏； 2）麦克风激活； 3）未受保护的移动设备管理（MDM）服务。这些漏洞是在运行iOS 11.1.2的越狱iPhone 5s上发现的，并已在运行iOS 12.0.1的非越狱第六代iPod Touch上的PoC应用程序中得到确认。

![](/images/2020-04-21/table7.png)

**File Provider State Dump:** File Provider daemons提供了一种方法为设备上运行的具有文件提供程序功能的应用程序回复状态信息（例如Google Drive，Microsoft OneDrive）。第三方应用程序可以三种方式滥用此泄漏的状态信息:
* 首先，泄漏的信息显示了已安装的具有文件提供程序功能的第三方应用。
* 其次，泄漏的信息显示了应用目录名称中使用的UUID，重新启动设备后这些名称也不会更改。因此，这些泄漏的UUID可以用于设备指纹（fingerprinting）识别。
* 最后，第三方应用程序可以推断“文件提供程序”目录中的文件名。iOS中有一个简单的辅助渠道，可让进程通过尝试读取文件的元数据来确定文件是否存在。如果该文件存在，则可能会发生权限拒绝错误或者正常读取元数据。如果该文件不存在，则会出现错误，指出该文件不存在。由于攻击者必须正确猜测文件路径，因此该side channel被文件路径中的UUID保护。但是，文件提供程序数据泄漏向第三方应用程序揭示了这些UUID，从而使它们可以推断文件提供程序目录中的文件名。通过使用有趣的文件名字典进行检查，可以加快这种推断。作为对文章披露的回应，苹果已通过CVE-2018-4446解决了此问题。

**Activate Voice Dictation:** 通过调用启动语音听写会话的方法，第三方应用程序可以在未经用户许可的情况下短暂激活麦克风（即用户未在其“隐私设置”中启用麦克风访问权限）。此方法导致铃声响起，表明麦克风已激活。使用未初始化的变量，应用程序无法访问音频记录，并且麦克风仅激活约1秒钟。作为对文章披露的回应，苹果已通过CVE-2019-8502解决了此问题。

**Inconsistent MDM Access Control Policy:** 在查看Kobold检测到的权利要求时，作者发现MDM管理服务的权利要求不一致。 `com.apple.managedconfiguration.profiled.public`mach端口提供了71种Kobold检测为可访问的方法。在这71种方法中，大多数需要与MDM相关的权利，但是其中21种方法没有明显的权利要求。对这21种方法进行手动调查发现了三种MDM服务，这些服务允许第三方应用程序禁用系统功能。即使受害者的设备尚未注册MDM，这些MDM服务仍然有效：
* 首先，可以禁止在所有移动浏览器上访问所有网站，并且尝试访问网站的用户将被要求输入未知的密码，其中记录了失败的尝试次数。
* 其次，可以禁用文本替换或键盘快捷键功能，并且在“设置”菜单中禁用用于配置新快捷键的菜单。
* 第三，可以禁用语音到文本功能的听写选项，并从“设置”菜单中删除启用该功能的切换。

### 6.2 Daemon Crashes

Kobold在运行iOS 11.1.2的越狱iPhone 5s和运行iOS 12.0.1的常规第六代iPod Touch上检测到crash。在表VIII中列出了检测到的崩溃，该列表根据堆栈跟踪分析列出了十个可执行文件，总共有14个unique crash。

![](/images/2020-04-21/table8.png)

**Crash Types:** 作者根据进程终止的方式将crash分为三种类型：
* abort signal：7个crash（3个daemons）在daemon发送abort信号时终止。该信号可能是断言某个值为空并响应而中止该过程的结果。
* segmentation fault：6个crash（6个daemons）当守护进程尝试访问零地址或零地址附近的无效内存地址时发生终止；作者推测这些异常的内存访问是由Kobold默认使用未初始化的变量作为远程方法参数引起的。由于segmentation fault暗示守护进程正在尝试使用损坏的值，因此作者认为其比中止信号崩溃更重要。作者所有的root权限守护进程崩溃都是由于segmentation fault引起的。
* killed by watchdog：Preferences进程冻结并且在闲置10秒钟后被watchdog进程杀死。

**Quantifying Crashes:** 作者通过两种方式量化崩溃，守护进程数量和唯一崩溃堆栈trace数量。作者开发了一个脚本来从这些崩溃报告中提取堆栈trace，并对其进行比较，以确定每个守护程序有多少个唯一的堆栈trace。例如，对于我们在com.apple.iap2d.xpc端口上调用的每个方法，accessoryd守护程序似乎都崩溃了。但是，堆栈跟踪分析显示，对端口的每种方法调用都触发了相同的堆栈，这意味着即使调用不同的方法，相同的问题也会导致崩溃。当wcd和replayd守护程序被不同的方法调用崩溃时，它们确实会生成唯一的堆栈跟踪，表明存在多个bug。

<!-- **Crash Causes:** 对于那些可重复发生的崩溃，作者隔离了导致崩溃的远程方法。在将崩溃报告添加到系统日志后，使用越狱设备上的脚本检测到了触发崩溃的方法，该脚本杀死了我们的方法调用应用程序。然后，我们在应用程序停止执行的代码行之前立即手动测试了这些方法。如果使用未初始化的参数值调用这些方法，则发现每种方法都会导致接收守护程序崩溃。请注意，willSwitchUser方法没有任何参数，但是如果由第三方应用程序调用，它仍会导致iTunes Store守护程序崩溃并出现分段错误。此iTunes Store守护程序崩溃表示一个错误，但是如果没有用于攻击者输入的字段，则它不太可能被利用。在运行我们的方法调用应用程序时，观察到其中有五次崩溃，但是它们的重复性不足以使我们为崩溃分配特定的方法。这些崩溃在表VIII中被标记为不确定的。 -->

<!-- **Inconsistent Entitlement Enforcement:** 在调查崩溃的方法时，我们注意到一种方法的权利执行不一致。重放提供了一个称为startRecording23的远程方法，如果该远程方法是我们的​​应用程序唯一调用的方法，则该方法将返回一条错误消息，指定必需的权利。但是，如果我们的应用程序在调用重播方法之前调用了一组十二个其他远程方法，则不会返回权利要求错误。取而代之的是，该方法触发提示，询问用户是否允许录制屏幕。如果用户接受提示，则重播将崩溃（崩溃可能是由于Kobold使用了未初始化的变量值）。这一发现表明，基于状态的条件（例如，先前调用的方法集）可能导致权利执行失败。 -->

## 7 Limitation

Kobold有两种类型的限制：
* 闭源系统固有的限制，无法量化Kobold可能无法检测到的方法的数量
* 尽管作者发现了代理混淆漏洞，但缺少访问控制策略规范来与其发现进行比较。
* iOS 11中添加了新的沙箱过滤器，但iOracle无法对其进行逆向。因此，可能有第三方应用程序可访问但未被此版本的Kobold检测到的mach端口。

**Argument Values:** 虽然Kobold可以静态提取远程参数方法的数据类型，但无法确定初始化的值。对于复杂的、无文档记录的class类型，其正确初始化需要进行大量的逆向或runtime的动态分析。

**Scope:** Kobold不检测共享库提供的NSXPC远程方法，也不检测NSXPC以外的远程方法的其他接口（例如XPC或Mach Interface Generator）。将静态和动态分析相结合的一般方法可能适用于这些接口，但是它们不适合静态分析（即classdump将无法检测其方法）。由于Xcode的错误阻止编译，因此有意从分析中删除了少量（少于十个）有问题的方法调用。 Kobold进行的entitlement调查不包括付费应用程序，仅分析了可用的免费iOS应用程序的样本。对包括付费应用程序在内的更多应用程序的纵向研究可能会发现新的semi-private权利。

**Black Box Testing:** Kobold依靠错误消息语义来推断远程方法的entitlement要求。但是，更复杂的分析方法（例如符号执行）可能会检测到Kobold遗漏的其他权利要求。 Kobold使用完成处理程序消息来确定成功调用了哪些远程方法。但是大量Kobold提取的远程方法没有完成处理程序。因此，确认远程方法调用（例如在守护进程代码中自动设置调试器断点）的灰盒测试形式可能会揭示更多第三方可访问的方法。

## 8 My Personal Opinion

本文的工作着眼于access control，不过这次Luke不像之前的SandScout和iOracle那样针对文件系统，而是针对XPC（准确来说是NSXPC）。但实际上这个工作难度不大，仅从entitlement入手，仅判断entitlement与服务的NSXPC之间的映射，也就是A-->B。

![](/images/2020-04-21/op.png)

但如何实现B-->C以生成exploit以及如何扩大scope是值得进一步研究的。