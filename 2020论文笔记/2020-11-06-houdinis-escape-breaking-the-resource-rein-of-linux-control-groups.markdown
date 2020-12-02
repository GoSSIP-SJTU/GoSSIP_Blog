---
layout: post
title: "Houdini's Escape: Breaking the Resource Rein of Linux Control Groups"
date: 2020-11-06 15:38:11 +0800
comments: true
categories: 
---

> 作者：Xing Gao^1^, Zhongshu Gu^2^, Zhengfa Li^4^, Hani Jamjoom^2^, and Cong Wang^3^
>
> 单位：^1^University of Memphis, ^2^IBM Research, ^3^Old Dominion University, and ^4^Independent Researcher
>
> 出处：CCS 2019
>
> 原文：https://gzs715.github.io/pubs/HOUDINI_CCS19.pdf

## Abstract

Linux Control Groups (cgroup) 是 container 技术中非常重要的机制，用于限制每个 container 的资源使用。cgroup 将进程分组，再用 controller 管理每个分组相应的系统资源，例如 CPU 和内存的使用。换言之，cgroup 可以计算每个 container 的资源使用量，并限制每个 container 可以使用的最大资源量。

每个新的进程在诞生的时候，都会继承父进程的 cgroup。因此 container 里新建的进程都会限制在同样的 cgroup 内。但是，有时候创建新进程时，并不能保证一致且公平的资源管理。这篇 paper 设计了几个策略脱离原本的cgroup 分组。当一些进程逃离原来的分组之后，这些进程所使用的资源就不会算在正确的 cgroup 里。

作者展示了五个 case study，都使得 Docker contaier 摆脱了原有的资源限制。在云计算容器平台上，恶意的 container 可以通过这样的方式大量消耗平台的算力或者其它系统资源，影响其它 container 的工作，DoS 攻击其它 containers。

作者在本地和 Amazon EC2 cloud 上分别进行了实验，证实了通过这样的方式可以比原先设置的限制多使用了 200 倍的 CPU，并且影响了同一台主机上其它 containers 的计算及 I/O 的 performance。

<!-- more -->

## Introduction

在 Linux kernel 中，有两个最重要的机制衍生出了我们现在使用的 container 技术，一个是 Linux namespaces，还有一个就是 cgroup。namespace 负责进程资源的隔离，而 cgroup 监管资源的使用。cgroup 防止 container 大量消耗 host 上的系统资源，避免一些 Denial-of-Service 攻击。

这篇 paper 系统地研究如何逃脱 cgroup 的资源限制，以及逃脱后造成的安全影响。新创建的进程会继承父进程的 cgroup，与父进程一起被限制在同样的 cgroup policy 下。作者设计了一套可以让进程逃离原本的 cgroup 的策略，进程使用的资源会被错误地计算在其它的 cgroup 中。

（**我觉得这个表述有点夸大其词，简单地说就是通过给 kernel 制造麻烦，给系统引入工作量，消耗主机大量资源。在这过程中，并没有使得攻击者拥有更多的可用资源。**）

作者在 Docker 环境上展示了五个实际的 case studies，都使得 Docker contaier 摆脱了原有的 cgroup 资源限制。作者分别利用了 kernel 的异常处理机制、数据同步机制、logging 系统服务、docker 引擎以及 softirqs 中断处理机制这五个机制实现的。

作者在本地和 Amazon EC2 cloud 上都进行了实验，发现即使在多个 cgroup controller 的控制之下，恶意的 container 依然可以大量地耗尽主机的资源，消耗 CPU，产生巨大的 I/O，并且这些行为都不会被计算在 container 的资源使用量上。

如果在云平台上，恶意的 container 会严重降低其它 container 的计算速度，并且在主机进行资源分配的时候获取一些优势。在实验中，恶意 container 可以比原先设置的限制多使用了 200 倍的 CPU，并且将同一个主机上其它 container 的计算及 I/O 的 performance 降低了 95%。


## Background
cgroup 机制将进程分成 hierarchical 的分组。用很多不同的资源 controller (或者叫 subsystem) 去限制、计算、隔离每个分组的资源使用。container 技术用 cgroup 限制每个 container。在云计算平台中，cgroup 可以计算每个用户的资源使用量进行计价收费。

### cgroups Hierarchy and Controllers

cgroup 是一个 hierarchical 的树状结构，每个计算任务在每个 hierarchy 中只能属于一个 cgroup，但可以属于不同的 hierarchy 中的 cgroup。每个 hierachy 会有一个或者多个 controller 负责 per-cgroup 的资源使用限制。

![cgroup](https://www.alibabacloud.com/forum/attachment/1609/thread/25_625_54477ce3606a8ac.png)

接下来介绍四种比较典型的 cgroup controllers。

#### cpu controller
cpu controller 会给每个 group 分配一个 share 表示该 group 的 weight。当多个 group 竞争 cpu 资源的时候，share 比较多的 group 可以获取更多的资源。如果有两个 container 在一个 core 上，第一个 container 的 share 是 512，另一个 container 的 share 是 1024，那么大体上第一个 container 会获得 33% 的 cpu 使用，第二个会获得 67%。

此外，cpu controller 还可以通过设置 quota 和 period 来控制 container 的运行时间。在一个 period 周期时间内，container 能运行的时间不能超过 quota，如果超过了 cpu controller 就会强制暂停运行直到下一个 period。如果设置 quota 为 50000，period 为 100000，那么这个 container 只会在一半的 CPU cycles 工作。

#### cpusets controller

cpuset 是指一组指定的 cpu core。cpusets controller 可以控制 container 的计算任务只在指定的一个或者多个 core 上运行。

#### blkio controller

blkio controller 控制 container 可以使用的 I/O 设备资源。这个 controller 和 cpu controller 比较类似，都是通过 weight 分配资源使用，管理每个 container 可以使用的 I/O 活动数量。

#### pid controller

pid controller 限制 container 内的 task 的数量。当进程数量超过了限制，pid controller 会阻止新的 fork 或者 clone。


### cgroups Inheritance

当一个新进程创建的时候，它会完完全全 copy 父进程的 cgroup。譬如父进程被 cpuset controller 限制住只能在第二个处理器上运行，那么子进程也只能在第二个处理器上运行。

## Exploiting Strategies

接下来我们介绍四种 exploiting strategies 摆脱 cgroup 的限制，以及背后的原理。

关键的 idea 就是，需要**将 workload 运行在并不是直接从发起请求的程序上 fork 出来的进程**。换言之，container 想要创建一个消耗系统资源的进程，但是这个进程并不是被认为是该 container fork 出来的子进程

这四个 strategies 都不需要 root 权限。从这张图里我们可以看到第一第二种利用了 kernel thread，第三种利用了系统服务，第四种利用了异常处理。

![figure1](/images/2020-11-06/uploads/upload_c7baea53869b39548d9efef60092875e.png)

### Exploiting Upcalls from Kernel

在 cgroup 机制里，所有的 kernel thread 都归属在 root cgroup，所有由 kernel thread 创建的子进程也都属于 root cgroup。因此，恶意的进程可以先让 kernel 创建一个 kernel thread 作为创建进程的 proxy，由 kernel thread 创建进程，这样新创建的进程和 kernel thread 一样属于 root cgroup，而非恶意进程所属的 cgroup。新创建的子进程就不受 cgroup controller 的限制了。

这个策略需要第一步 user space 的进程触发 kernel space 的 kernel function (可以通过调用 system call)， 第二步从 kernel space 调用执行 user space 的进程（usermode helper API）

### Delegating Workloads to Kernel Threads

第二种利用 kernel thread 的策略是将 workload 安排给 kernel thread，让 kernel thread 使用更多的资源，而资源使用也会计算在 kernel thread 的用量上。

Linux kernel 平时会运行很多的 kernel threads 各司其职（例如 *kthreadd*, *kworker*, *ksoftirqd* 等等等等）。而常常有研究表明，这些 kernel thread 会因为一些 bug 消耗大量大量的资源。如果进程可以让 kernerl thread 跑一些 workload，就能摆脱 cgroup 的限制。

### Exploiting Service Processes

除了维护 kernel thread 以外，Linux 还会跑一些系统进程 (如 *systemd*) 用于进程管理、logging、debugging 等等之类的工作。这些系统进程监管其它进程并执行相应地任务，而且有时候会依赖普通的 user space 的进程。

如果可以让这些系统进程大量工作，那资源使用会被计算在系统进程所在的 cgroup，而不是用户进程的 cgroup。

### Exploiting Interrupt Context

最后一个策略是利用 interrupt context 的资源消耗。cgroup 的机制只会计算在 process context 里的资源使用情况，一旦 kernel 跑进了别的 context (如 interrupt)，那所有的资源使用都不会计算在任何 cgroup 里。

所以如果可以大量地触发 interrupt，那也不计算在进程的 cgroup 的资源使用内。

## Case Studies on Containers

上面讨论了四种理论上的 strategies，但在实际操作中，由于各种各样其它的安全防护机制，实践难度比较大。但作者还是在 docker container 中基于上面四条策略设计出了五个可行的 case。


#### Threat Model

在多租户容器环境中，不同租户拥有的不同 Docker container 共同一台物理主机。管理员利用 cgroup 设置每个 container 的 CPU 时间、内存、I/O 带宽的资源限制，并且固定在特定的 core 上运行。

我们假设攻击者控制了一个 container，并尝试利用 cgroup 的不足之处来实现两个目的：（1）降低其他 container 的性能（2）在资源分配的竞争中获得不公平的优势

#### Configuration
![table1](/images/2020-11-06/uploads/upload_e510b3bcb520467f2a10678e28ae986c.png)


#### Result summary
![table2](/images/2020-11-06/uploads/upload_6e0400081359dc9d35dfe9328a0dc007.png)



### Case 1: Exception Handling

第一个 case study 利用了内核中的异常处理机制 (strategy 1)。在异常处理的过程中可以调用 usermode helper API 触发 user space 的进程。

#### Detailed analysis

在异常发生的时候，进程会被终止，并且会调用 *core dump* 内核函数生成用于 debug 的 *core dump* 文件，而 *core dump* 函数会通过 usermode helper API 调用 user space 的 *core dump* 应用（在 Ubuntu 上默认的应用是 Apport）。Apport 是由 kernel thread 创建的，而不是 container 创建的，因此 Apport 不受 container 的 cpu/cpuset cgroup controller 限制。不断地抛异常，Apport 就会不断地工作，消耗系统资源。

#### Evaluation

在 container 内写一个循环 div 0 的程序不停地抛异常。container 被限制在一个特定的核上运行，而且 cpu 使用量以及可创建进程数量也都受 cgroup 的限制。实验统计 container 的 CPU 资源使用情况以及整台主机的 CPU 资源使用情况。

![figure2](/images/2020-11-06/uploads/upload_0047d0b258bf60aada3efdaf8972c45d.png)

1200% 代表着 12 个 core 全都 100% 的使用率。尽管 cpuset controller 限制 container 只能在一个 core 上运行，但是这个限制完全被打破了，因为处理异常的程序可以运行在别的 core。我们可以看到，系统认为 container 使用的 CPU 资源都非常合理，但异常处理机制将整台主机的 CPU 资源都消耗殆尽。

通过不断地生成异常，container 可以消耗比限制多 200 倍的 CPU 资源，从而将同一主机（不限于一个内核）上其他 container 的性能显著降低 85％ 至 95％。

PID controller 通过限制 container 内的进程数量来减轻 cgroup escape 的严重性，但即使 container 只能有 50 个进程，依然可以达到 144X 的资源放大。

![table3](/images/2020-11-06/uploads/upload_66f93fd30af5e4cd0c272a32aab5ec07.png)

而和恶意 container 运行在同一台主机上的其它 container 受到 DoS 攻击。因为系统资源全被消耗光了，它们能使用的 CPU、内存、或者 I/O 都大大地降低（至少 80%）。


### Case 2: Data Synchronization

第二个 case study 是利用磁盘数据同步的 writeback 机制 (strategy 2)。在没有任何其它操作的情况下，只有当 cache 上的数据被 evict 的时候，数据才会写入磁盘，但可以通过调用相应的 syscall 手动同步 cache 和硬盘上的数据。由于 writeback 机制中发启同步操作的进程和启动 I/O 的进程与是分离开来的，因此我们可以利用数据同步逃脱 cgroup。

触发数据同步的方法有多种，作者用了 *sync* syscall，*sync* 可以将 cache 内所有修改 writeback 回文件系统）。

#### Detailed Analysis

调用 sync 的过程中，sync 创建了一个 kernel thread。这个 kernel thread 扫一遍整个文件系统（不单单是 container 里的文件系统），将所有的 dirty cache 都写回硬盘。因此我们可以利用同步来降低系统的 I/O 性能。

#### Evaluation
在实验中，恶意的 container 不停地调用 sync，并与其它 container 限制在不同的 core 上运行。实验统计其它的 I/O-intensive 的 container 的性能被影响的严重程度。

![figure3,4](/images/2020-11-06/uploads/upload_f5a30d3064dcbeb0e5bfc8127afde8dc.png)

我们可以看到可以用这种方式进行 I/O-based DoS 攻击。

同时，在系统资源的竞争中，如果其它 container 由于 I/O 操作过慢而影响运行速度，那 victim container 的 CPU 使用量就会减少，那么恶意的 container 就可以在竞争中获取更多的资源。

### Case 3: System Process - Journald

第三个 case study 利用了 systemd-journald 系统服务(strategy 3)。systemd-journald 用于收集系统内的 log 数据。

#### Detail Analysis

系统服务进程有自己的 cgroup，并不属于 container 的 cgroup。

在默认情况下 container 里的很多操作是不会 log 的，但有三类特定的操作即使在 container 内部触发也会 log：切换 root 用户，添加新的 user/group，抛出异常

触发 logging 也会给系统的 I/O 带来负担。

#### Evaluation

恶意的 container 不停地用上面三种方式触发 log，并且被限制在与其它 container 不同的 core 上运行。

![figure6](/images/2020-11-06/uploads/upload_5fa8c38a82271b3a459a5292aca5d5d9.png)

可以造成 5%-20% 额外的 CPU 使用以及平均 2MB/s 的 I/O throughput。在服务器上，其它 victim container 会被拖慢 1,000 IOPS。

### Case 4: Container Engine

第四个 case study 利用了 container engine (Docker) 在 kernel thread 和 container engine 上触发 workload。

作者利用了 tty device (terminal) 给 Docker 以及 kernel 增加了工作量。

#### Detailed Analysis

container 和 tty device 交互的时候，数据会经过 docker 的 CLI 进程和守护进程，最后达到 tty 驱动。而 tty 驱动由 kernel thread 执行。因此，只要 container 不停与 tty device 交互，就能让 docker 和 kernel 消耗不少的资源而不计算在 container 的 cgroup 里。

#### Evaluation

恶意的 container 不断地让 terminal 显示主机上所有的已加载的模块，且被限制只能在一个 core 上运行。

![figure7](/images/2020-11-06/uploads/upload_8df0bec83523c4c708483d59d899a5f5.png)

通过这种方式，container 可以给主机造成 300% 的 CPU 资源使用，其中大部分都被视为 docker 和系统消耗的。

### Case 5: Softirq Handling

第五个 case study 利用了 softirq (software interrupt request) 中断处理机制中的 NET softirq。

#### Detailed Analysis

可以利用 NET softirq。当网络包传送结束的时候，就会发启 interrupt。在一般情况下，处理 interrupt 的 overhead 可以忽略不计，但是在 iptables (防火墙系统) 的存在下，iptable rule 数量越多，系统处理网络中断的计算量就越大。

Docker 会给每个 container 设置一套防火墙规则用于网络隔离。有时一个 container 里会用多个规则，甚至有些情况下 container 可以设定系统的防火墙规则。

#### Evaluation

实验测量了在不同数量的 iptable rule 情况下的所有 interrupt 处理机制的 CPU 使用。

![figure8](/images/2020-11-06/uploads/upload_1443bf8045ddfc63f7b5bdc2b7785e65.png)

当有 1w 个 rules 的时候，overhead 大约 40%。

## Mitigation Discussion
作者提出了几个比较 intuitive 的解决方案
1. 需要有更加细粒度的机制 track 资源消耗的源头
2. 开发更多新的 cgroup controller (eg. logging) 
3. 对一些敏感操作（eg. sync），设置 container 可使用次数限制

## Conclusion

这篇文章分析了 cgroup 在判定资源使用者时的不足，提出了一系列打破 Linux Control Group 资源限制的方式，并在实际的 Docker 环境下，大量消耗主机资源，对其它 victim containers 进行了攻击。




