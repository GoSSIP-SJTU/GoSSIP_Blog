---
layout: post
title: "Security Namespace: Making Linux Security Frameworks Available to Containers"
date: 2019-06-27 13:32:05 +0800
comments: true
categories: 
---
作者: Yuqiong Sun, David Safford, Mimi Zohar, Dimitrios Pendarakis, Zhongshu Gu and Trent Jaeger

单位: Symantec Research Labs, GE Global Research, IBM Research and Pennsylvania State University

出处:  USENIX Security Symposium 2018

原文: https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-sun.pdf

<hr/>

轻量级虚拟化（即容器）无需使用单独的内核即可为应用程序提供虚拟主机环境，实现更好的资源利用率和效率。但是，共享内核会妨碍容器利用传统VM可用的安全功能，无法应用本地策略来管理完整性度量，代码执行，强制性访问控制等。在本文中，作者提出了一种内核抽象security namespaces，它能使容器自主控制其安全性。作者提出了一种可以向容器动态的分发操作的路由方案，保证由一个容器产生的安全策略不会影响主机或其他容器。最后展示了在Docker上使用针对内核的完整性度量和强制访问控制的security namespace。结果表明，security namespace可以有效地缓解容器内部的安全问题，系统调用的额外消耗小于0.7％，应用程序吞吐量几乎相同。因此，security namespaces的实现可以使容器获得自主控制的能力并且不会破坏主机和其他容器的安全性。

<!--more-->

## 背景
### Namespace 和 容器

![](/images/2019-06-27/media/15613482765212/15613502442188.jpg)

事实上，还有Cgroup，一共7种类型。

![](/images/2019-06-27/media/15613482765212/15613503479583.jpg)

容器可以为一组进程创建一个隔离的运行环境。如图是Docker创建容器的流程。

### Kernel Security Frameworks

内核中有许多安全框架，包括Linux integrity subsystem,  SELinux, AppArmor。这些框架都有相同的设计原则，即在内核中添加“Hook”的代码，根据实现制定的策略拦截来自进程的安全关键操作。

Linux integrity subsystem, 即 Integrity Measurement Architecture, 旨在阻止Linux系统上的文件的更改，特别是可执行文件。IMA通过测量可能影响的文件来实现系统的完整性。使用TPM等安全协处理器，IMA可以安全地测量然后向第三方提供可信证明。除了认证，IMA也可以指定可以加载的文件来保证执行系统的完整性。

## Motivation
### 自主安全控制
理想情况下，当用户的应用部署在VM或主机上，安全控制应该是自主的。然而，已有的内核安全框架很难实现自主控制。例如，考虑部署在公共云上的容器化银行服务。服务所有者想要确保关键服务组件（如服务代码，库）和配置不会被修改。但是，银行服务不能使用IMA证明其完整性。原因是IMA作为内核安全机制，跟踪整个系统的完整性。因此，对于不同的容器（和主机系统）混合在一起，无法独立访问。第二，银行服务无法控制哪些代码或数据可以被装入容器。由于IMA只允许一个单一策略制定者即云厂商，个别容器不能决定要度量哪些文件。Linux内核中的安全框架被设计为全局的、强制性的。只有所有者（即系统管理员）被授权设定策略。系统上的其他principal（即容器所有者）不得做出安全决定。要使容器具有自主安全控制，就需要放宽全局和强制性假设的安全框架。

### Security Namespace
实现自主安全控制的一个想法是设计一个Security Namespace，类似如何隔离/虚拟化Linux操作系统。 但是，与其他namespace不同，Security Namespace需要放宽系统的全局性和强制性假设。作者首先介绍一种Security Namespace设计的草稿，并提出两个攻击示例。

#### Strawman design
![](/images/2019-06-27/media/15613482765212/15613534733659.jpg)

直观的设计是将内核安全框架（即，通过复制代码和数据结构）虚拟化为虚拟的实例。每个虚拟的实例都成为security namespace：它与一组进程相关联，它独立地对这些进程实施安全决策。例如，如图所示，P0进程在NS native中运行，它创建新的namespace NS1并自行创建（即通过CLONE NEW创建）。子进程P1现在在NS1中运行， P1进一步在同一namespace中创建，P2在新安全性中进一步创建P3的命名空间。在这种情况下，将P0的安全控制分配给NS native，P1和P2给NS1，P3给到NS2。虽然这种设计以直接的方式实现了自主安全控制，但它引入了两种攻击：

#### 攻击场景1
![](/images/2019-06-27/media/15613482765212/15613534547038.jpg)
假设NS native和NS 1都是基于IMA namespace的，即native system的所有者可以通过NS native度量和记录所有执行的代码来证明其完整性。然而，恶意的进程P可能会通过fork创建一个新的IMA namespace NS1并在其中执行，此时恶意进程的度量存在NS1的list中，在NS1退出时即被删除，native namespace将无法度量NS1的代码执行，最后证明native系统是完整可信的。

#### 攻击场景2
一个容器NS1向另一个容器NS2共享文件。NS1使用read only的方式保证文件的完整性。与此同时，NS2使其进程对该文件有可读可写的权限。如果NS2对文件做了修改，并且NS1依然信任该文件，就会破坏NS1的假设。这个例子说明，两个主体独自做安全决策时，两者产生的不一致性可能会打破某些安全框架的假设。

### 目标
通过自主安全控制，各个security namespaces可以管理自己的安全性。 具体来说，设计具有以下三个属性：

* 与security namespace关联的进程将受到该namespace的安全控制

* 拥有security namespace的主体可以定义该namespace的安全策略，独立于其他security namespace和本机系统。

* 安全状态（例如，日志，警报，度量和等）是独立维护和访问的。

### 安全模型
首先假设内核是可信的。安全框架及其namespace在内核中实现，它们也是可信的。所有的用户空间进程都是不可信的，他们是 security namespace的限制目标。实践中，通常有某些用户空间进程负责将安全策略加载到内核中，这些进程也不受信任。内核保证策略的完整性，因此只有正确合法的策略才会被接受。另外，假设系统中的主体之间不存在相互信任。Security namespace抽象的设计目标是防止一个主体破坏另一个主体的安全。

## Solution Overview
前面的方案告诉我们，当放宽全局和强制的安全假设时，我们要考虑系统上的所有的主体，（考虑一致性）。归结为两点原则

* 对于一个进程的操作，所有的security namespaces都了解其操作（即通过其安全策略表示）（注，不一定需要了解所有的操作，只需要了解与其安全有关的操作）

* 仅当所有security namespaces都表达了意见后，才由系统决定操作。

作者提出了基于路由缓解的设计方案。当进程执行一个操作（比如system call），先会发给Operation Router，并由路由决定哪个security namespaces应该知道这个操作。每个security namespaces独自做出决策。最后由system汇总所有决策做出最终决策。

由于securitynamespaces不能查看其它的namespace的状态，所以设计了Policy Engine用来确保policy load的时候，policy互相之间不会冲突。

![](/images/2019-06-27/media/15613482765212/15613584473536.jpg)

## Operation Router
Operation Router识别哪些security namespaces需要对某个操作表达意见并发送给他。 这里基于一个观察，security namespaces可能对一个操作有表达意见，如果Operation Router没有将操作路由到security namespaces，那么全局和强制的安全假设就可能会被破坏。一个操作可以写成元组（s，o，op）的形式，下面分别从Subject和Object的角度进行讨论。

### A Subject’s Perspective
```
Security framework makes an implicit assumption about
its globalness: it controls all subjects on a system that
are stemmed from the very first subject that it sees.
```
安全框架隐式的表达了其全局性。所有subject都是第一个subject的继承者。比如，所有的进程都是从第一个进程init fork而来的。在第一个攻击场景中，P1是P的后代，然而当P1创建新的namespace后，NS native不再限制P1，也就打破了NS native的隐式的全局性的特性。
因此，如果一个subject的操作没有被路由到应该去的security namespace，就会打破全局性。所以，当需要路由时，不仅要路由给和该subject相关的namespace，还有路由给其直接的祖先。

### An Object’s Perspective
```
Security policy is often a whitelist, enumerating allowed
operations from subjects over objects.
```
安全框架强制控制意味着除了被允许的操作，其他操作都不能越过object。在第二个攻击示例中，即打破了强制性的假设。因为在共享文件时，一个NS1假设其只读，另一个NS2假设其可读可写。当NS2中修改了文件，NS1对此一无所知。
因此，Operation Router需要考虑通过将操作路由到所有subject可能访问的安全名称空间以确保他们所有的安全。

### Shared Objects and Authority
理想情况下，每个security namespace都与其独立的一组object的resource namespaces相结合。实际上，某些object可以被多个security namespaces访问。比如proc和sys文件系统，这些object可以在不同的容器中共享。这就会导致两个问题。首先，由于安全策略的白名单性质，security namespace只允许自己的允许的操作，并拒绝来自其他security namespace的操作。第二，如果Operation Router将一个security namespaces操作路由到另一个security namespaces，因为它们共享访问权限一个对象，它可能会导致隐私泄露。

为了解决这一问题，引入两个概念authority和external。如果一个security namespace被标记为authority，那么所有相关操作都会强制路由到所有的namespace。
external用来和authority一起使用，当security namespace声明一个object是 authority的时候，可以定义策略对于其他namespace的不可见的，这时就要用external来修饰。

## Policy Engine
设计policy engine是为了检测策略之间是否冲突。有两种类型的冲突

*  DoS conflicts
当security namespace加载策略时，如果一个subject的操作可能被另一个namespace拒绝，就叫DoS conflicts。

*  expectation conflicts
当security namespace加载策略时，如果该策略可以拒绝来自其他安全的操作
命名空间，称之为expectation conflicts

![](/images/2019-06-27/media/15613482765212/15613599718132.jpg)


## Implementation
用内核安全框架 IMA 和 AppArmor，分别改动了大约1.1K和1.5K代码。

为了创建新的namespace，修改了clone和unshare两个syscall，扩展了CLONE_NEWIMA 和 CLONE_NEWAPPARMOR。

IMA和AppArmor通过securityfs 接收策略和导出security state。

## Evaluation
### IMA Namespace
有效性

首先测试自主安全控制，使用3种不同类型的恶意代码

* 没有签名的code
* 未知key签名的code
* 修改了签名的code

第二个测试安全性
试图添加允许任意程序执行的策略，被Policy Engine阻止。

### AppArmor Namespace
container不能利用AppArmor namespace破坏host，因为冲突的操作最终会被system阻止。并且，在策略加载时Policy Engine会通知container使得container因为资源访问错误而不会运行。

### 性能
Dell M620 server 2.4Ghz CPU 64GB 
Ubuntu 16.10 kernel version 4.8.0
![](/images/2019-06-27/media/15613482765212/15613610600578.jpg)

![](/images/2019-06-27/media/15613482765212/15613610260955.jpg)
