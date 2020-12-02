---
layout: post
title: "SPEAKER: Split-Phase Execution of Application Containers"
date: 2019-05-29 16:44:34 +0800
comments: true
categories: 
---
作者: Lingguang Lei, Jianhua Sun, Kun Sun, Chris Sheneﬁel, Rui Ma, Yuewu Wang, and Qi Li

单位: Institute of Information Engineering, Chinese Academy of Sciences, Beijing, China

出处:  DIMVA'17

原文: https://link.springer.com/chapter/10.1007/978-3-319-60876-1_11

<hr/>

由于syscall是进入内核的进程的入口点，Linux seccomp filter已集成到流行的容器管理工具（如Docker）中，以有效地约束容器可用的syscall。但是，Docker缺少一种方法来获取和定制给定应用程序的syscall集合。在本文中，作者提出了一种名为SPEAKER的容器安全机制，它可以通过在两个不同的执行阶段（即启动阶段和运行阶段）定制和分析其所需的syscall来显着减少对给定应用程序容器的可用系统调用数量。

<!--more-->

## 背景
### Linux Namespace
namespace用来隔离系统资源，如pid, user, uts, mnt, net, and ipc

### Seccomp
Seccomp用来限制syscall调用，可以减少进入kernel的入口。

docker使用`run --security-opt seccomp`来加载seccomp的配置，只在启动的时候生效，当前docker默认允许313个system call。

工作模式：
* seccomp-disabled
* seccomp-strict
    只能使用read、write、_exit和sigreturn
* seccomp-filter
    允许设置白名单或者黑名单
    必须设置no_new_privs，即通过fork、clone以及execve产生的进程继承父进程的权限。

缺点，只能设置当前进程的seccomp filter。

### 设计和实现

SPEAKER包括Tracing和Slimming两个模块。

![](/images/2019-05-29/media/15589244840060/15589300476762.jpg)

* Tracing Module
    定制booting和running阶段用到的syscall

首先为什么要划分两个阶段？作者考虑到boot阶段虽然很短，但他需要一些特别的syscall来初始化执行环境，而这些syscall在running的阶段就不太可能使用了，对于running阶段同样如此。

 ![](/images/2019-05-29/media/15589244840060/15589302525980.jpg)

那么如何区分这两个阶段。基于轮询的方式，通过不断检查相应的service的状态，当booting阶段结束，service变成running的状态。大多数service，比如apache，mysql，nginx等。基于经验的粗粒度分离，一个是boot阶段很短，可以在10s之内，另一个在running阶段，syscall的数量的变化接近平缓，那么就可以定一个阈值。

![](/images/2019-05-29/media/15589244840060/15589313207651.jpg)

然后需要tracing system call，可以对image静态分析，也可以在执行的过程中动态分析。作者直接用了strace，因为容器内的进程和host的进程存在一一映射的关系，从host对相应的进程strace即可。

* Slimming Module
    限制booting和running阶段用到的syscall

首先介绍一下seccomp filter的机制。一个进程可以存在若干filters，它们被一条单链表连接起来。
bpf_prog结构体里每条指令由四元组组成。

如图，第一条是read(), write(), rt sigreturn() and exit()。
第二条是read()， write() 。
![](/images/2019-05-29/media/15589244840060/15589325998732.jpg)

这里存在一个问题就是，一旦某个syscall被禁止了之后，对于这个进程来说，即使添加了新的filter也不可能再使用这个syscall了。因此为了解决这个问题，作者实现了一个内核模块，去修改bpf_prog指针。当更新了一个进程的bpf_prog, 所有的子进程都会改变。

![](/images/2019-05-29/media/15589244840060/15589325891169.jpg)

## 实验结果
### System Call Reduction
Web Server Containers

![](/images/2019-05-29/media/15589244840060/15589359247368.jpg)

Data Store Containers
![](/images/2019-05-29/media/15589244840060/15589359153004.jpg)

### Performance Overhead
![](/images/2019-05-29/media/15589244840060/15589355686931.jpg)

* tracing module 
    bash script 570 SLOC

* slimming module
    user level code with 2256 SLOC
    kernel module with 1088 SLOC