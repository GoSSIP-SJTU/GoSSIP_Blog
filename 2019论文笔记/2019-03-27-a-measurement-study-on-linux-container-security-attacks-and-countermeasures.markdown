---
layout: post
title: "A Measurement Study on Linux Container Security: Attacks and Countermeasures"
date: 2019-03-27 19:21:44 +0800
comments: true
categories: 
---
作者: Xin Lin, Lingguang Lei, Yuewu Wang, Jiwu Jing, Kun Sun, Quan Zhou

单位: University of Chinese Academy of Sciences

出处: ACSAC'18

原文: https://dl.acm.org/citation.cfm?id=3274720

<hr/>

近年来，随着docker逐渐被大众认可，越来越多的人开始担忧容器的安全性。本文作者通过测试了88个Exploit-db上的漏洞，尤其是针对内核漏洞的提权攻击，系统的评估了docker在默认配置下的安全特性，发现56.82%的漏洞在容器下仍然能被成功利用，并进一步分析了容器的安全特性对内核提权漏洞的影响。

<!--more-->
## 背景
### Linux Container
进程隔离，依赖内核的特性
* namespace（user, uts, net, pid, mnt, ipc and cgroup）
* Cgroup

### Linux Kernel Security Mechanisms
* 自主访问控制
    * Capability (38个功能)
    * Seccomp
* 强制访问控制
    * SELinux
    * AppArmor
    
### CPU Protection Mechanisms
* KASLR
* SMEP
* SMAP

## 攻击数据集
### 收集Exploits
Exploit-db
*   web application
*   remote, local & privilege escalation
*   denial of service
*   poc (Proof of Concept)

2017年10月20号，作者收集了2016年和2017年提交到Explot-db上的400个exploit。
274个CVE是2016-2017公布的，24个2013-2015年公布的。

去掉一些GUI软件（浏览器、Flash）后，保留了223个exploit。

### 分类
作者认为Exploit-db上的分类不科学，比如web和DoS有交叉。所以作者提出了两个分类方法。

按组件分
* web application
* server
* library
* kernel

按结果分
* 信息泄露
    *   构造恶意的payload
    *   条件竞争
    *   绕过MAC
* 远程控制
    * Injection
        * XXE
        * XSS
        * SQLi
    * 构造request/reponse
        * CSRF
        * SSRF    
* 拒绝服务
    *   资源耗尽
    *   disability（panic或者crash）
* 权限提升
    * 内存修改
    * 文件修改 

### Exploit数据集
一个exploit可能对应多个漏洞，一个漏洞也可能对应若干个exploit。
经分析223个exploit，包含148个漏洞。

其中web app占59.19%，lib layer大多是DoS类型的漏洞。76.09%的内核漏洞可以用来提权。68.63%的提权是因为内核漏洞造成的。

![-w462](/images/2019-03-27/media/15535084178673/15535161976236.jpg)

## 安全评估
### 实验准备
比较原有平台和docker里的攻击结果，分为用户层和kernel层。除去难以配置环境的，233个exploit里共有88个被用来做实验。然后在实验中，又有41个exploit被剔除，有的是因为旧版本获得不到了，有的是因为exploit不能成功运行。另外，对于类似的漏洞（除了提权），作者尽量保留一个。
![-w453](/images/2019-03-27/media/15535084178673/15535168140613.jpg)

### 结果概述
88个exploit里56.82%在容器里仍有效，并且大多是用户层的。由此可见，容器对于其内运行的程序没有过多的安全加强。

![-w404](/images/2019-03-27/media/15535084178673/15535172761889.jpg)

* 信息泄露
web和server的信息泄露HTTP请求（比如CSRF）造成，所以没有被容器的安全策略阻止。
kernel的被阻止是因为调用eBPF API需要CAP_SYS_ADMIN能力。

* 远程控制
容器的安全策略不能阻止用户层的XSS、XXE、SQLi、命令注入等攻击。

* 拒绝服务
所有的server和lib层的拒绝服务都没有被阻止，这是因为docker默认没有使用Cgroup策略（即namespace和host一致）。如果加了Cgroup后，有5个可以被阻止，其余的因为kernel panic和crash导致不能被防御。
4个kernel层的被阻止是因为Capability不足或者是Seccomp阻止了特定的syscall。

* 权限提升
提权失败的主要原因是Capability不足，docker默认只提供了14个Capabilities。

### 提权攻击
容器最大的安全隐患就是共享内核，因此提权可以打破容器提供的隔离能力。作者通过逐个分析这些exploit，来评估容器的安全特性。

![-w233](/images/2019-03-27/media/15535084178673/15535182297729.jpg)

* CPU Protection Mechanisms
由于KASLR，攻击者一般要先leak kernel的指针，进而计算实际的kernel里函数的地址。一个方法是使用demsg查看系统boot时内核打印的日志。
由于SMEP和SMAP，内核不能直接访问用户态的数据和代码。一般是使用native_write_cr4把相应的位置置0。

* Container Mechanisms
Namespace和Cgroup阻止了21.62%的攻击。比如CVE-2016-5195和CVE-2016-4557因为mnt namespace失败，因为/etc/passwd和/etc/crontab 是只读的。攻击者只能改变容器内的，改变不了Host系统内的文件。

* Kernel Security Mechanisms
其他的内核安全特性Capability, Seccomp和MAC阻止了86.49%提权攻击。大多是因为缺少CAP_NET_ADMIN能力。

    * Capability
因为缺少某些特殊的Cap导致调用API失败，比如CVE-2017-6074调用setsockopt()时设置IPV6_RECVPKINFO需要CAP_NET_ADMIN的能力。
因为cap_bset结构体决定了一个进程最多能或者的能力，而docker默认是给了14个能力，而root具有38个能力，因此导致失败。比如因为容器内的具有root权限的程序被利用导致普通用户提示权限的，那么他最多能获得14个能力。（如果拿到内核权限是有办法拿到所有的能力的）

    * Seccomp
      CVE-2016-8860因为mount失败
      CVE-2016-0728因为keyctl失败
      CVE-2016-5195因为ptrace失败
    
    * MAC
      由于AppArmor和SELinux导致mount某些文件系统失败
  
* Relationships Among the Security Mechanisms
这些安全特性互相也是有关联的。这里作者表达的意思应该是如果存在任何一个安全特性被误用的情况，也可能导致提权成功。

    * 一些exploit只有被namespace和capability同时作用才能起效。
    * 一些kernel函数可以通过设置Capability或者Seccomp来开启。
    * 合理的设置Seccomp规则或者MAC策略可以阻止一些exploit。
    * 合理的设置Namespace或者Cgroup配置可以阻止一些exploit。

### 结论
* 对于容器内的进程，除了用于提权的程序外，容器本身没有提供足够的安全加强
* 默认的Cgroup策略不能有效的防止DoS攻击
* kernel的安全缓解措施Capability, Seccomp和MAC对于缓解提权攻击起到了一定的作用
* 不合理的配置可能会导致安全缓解措施提供的能力降低

## 提权过程

作者先是总结了在容器里提权的一般攻击模型，然后提出了一个防御系统。

### kernel提权攻击模型
在docker默认的配置下，只有4个exploit利用成功。其他的7个被因为没有“CAP_NET_ADMIN”权限而失败。
但是作者还是根据这11个exploit总结了提权的4个步骤。

* bypass KASLR
* 触发漏洞（UAF、条件竞争、缓冲区溢出）
* ROP（关闭SMEP&SMAP）
* ret2usr

![-w860](/images/2019-03-27/media/15535084178673/15535115919660.jpg)

### 防御措施
作者这里考虑从调用commits_creds时增加检查来防御提权攻击。

commits_creds调用时的相关过程如下。
![-w437](/images/2019-03-27/media/15535084178673/15535116123720.jpg)

防御方法，修改了*commits_creds*函数，在更新*real_cred*和*cred*时，检查caller是否在容器内。检查的方法也很简单，*task_struct*结构体里有个*nsproxy*指针，保存着对应进程的*namespace*信息，那只要比较一下是否和*init_nsproxy*一样。（攻击者先改掉nsproxy不就好了？）。这里作者说只要加不到10行代码。

### 有效性和性能
11个exploit能有效的被防御。在Ubuntu 16.04 2.8GHz 2G内存的虚拟机上测试，效率如下。
![-w439](/images/2019-03-27/media/15535084178673/15535121386125.jpg)
