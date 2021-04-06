# POP and PUSH: Demystifying and Defending against (Mach) Port-Oriented Programming

> 作者：Min Zheng∗, Xiaolong Bai∗, Yajin Zhou†⋆ , Chao Zhang‡§ and Fuping Qu∗
> 
> 单位：∗ Orion Security Lab, Alibaba Group，† Zhejiang University，‡ Institute for Network Science and Cyberspace, Tsinghua University，§ Beijing National Research Center for Information Science and Technology
> 
> 会议：NDSS 2021
> 
> 论文链接：[https://www.ndss-symposium.org/wp-content/uploads/2021-126b-paper.pdf](https://www.ndss-symposium.org/wp-content/uploads/2021-126b-paper.pdf)

## 摘要

这篇文章研究的对象是iOS、macOS这类基于XNU的操作系统。针对XNU kernel的传统攻击通常同时需要信息泄露和控制流劫持这两类漏洞，即先借助信息泄漏漏洞来解决KASLR，然后借助控制流劫持漏洞去执行ROP gadget。然而，由于PAC【之前我们有推荐过相关论文】在苹果设备的部署，控制流劫持变得很困难。于是也就有了本文总结的(Mach) Port Object-Oriented Programming (POP)攻击与对应的PUSH防御。

<!-- more -->

## 1 背景

### 1.1 iOS Jailbreak & Mitigations

目前越狱主要分为以下两类：

1. 以硬件漏洞为基础的 **BootROM Exploit**：攻击secure boot chain，只能通过更新硬件来解决。例如[***checkm8***](https://github.com/axi0mX/ipwndfu)及基于它开发的越狱工具[***checkra1n***](https://checkra.in)
2. 基于软件漏洞的 **Userland Exploit：**将任意数据写入内核的堆，使之成为有效的 Kernel 数据结构，进而从用户态实施对内核的非法控制。
    - 由于用户态的应用程序无法获取 kernel_task，因此需要采取间接操作内存的方式，如Socket、Mach Message、IOSurface等。
    - UAF这类内存漏洞利用过程中，需要间接控制系统对堆上内存的 Reallocation。常见的方式就是 ***堆喷([Heap Spraying](https://en.wikipedia.org/wiki/Heap_spraying))*** 和 ***堆风水([Heap feng-shui](https://en.wikipedia.org/wiki/Heap_feng_shui))*** ，通过往堆上放置很多与目标内核对象大小相同的伪造对象来传入内核进行控制。

目前的防御措施与对应手段：

1. Data Execution Prevention (DEP)：采用code-reuse attack，如ROP；
2. Kernel Address Space Layout Randomization (KASLR)：通过内存泄露漏洞来获取内存中的gadget；
3. Control-flow integrity (CFI)：尤其是从A12芯片开始引入的Pointer Authentication Codes（PAC）这种借助硬件辅助的控制流完整性保护，使得控制流劫持变得困难。应对方法是通过PAC实现的缺陷去bypass（参考Brandon在MOSEC和BlackHat的演讲）。

High level的jailbreak路线：

1. 获取tfp0: `task for pid()` 是一个让被授予权限的进程获取同在主机上另一个进程的函数。仅获取tfp0 只能进行 kread, kwrite 等基础操作，要实现 rootfs read/write, kexec还要其他步骤。
2. 通过 tfp0 实现沙箱逃逸，实现 rootfs 读写
3. 定位内核数据（如String XREF）
4. 通过PACIZA Signing Gadget来绕过PAC
5. 实现内核任意代码执行（如IOTrap）

### 1.2 Mach Port & Task

Mach ports 是内核提供的进程间通信机制。Mach port 在用户态以 mach_port_t 句柄的形式存在，以Mach port name来表示，并且在内核空间中都有相对应的内核对象 ipc_port，如图1所示。它相当于一个受内核保护的单向管道，它可以有多个发送端，但只能有一个接收端（处理系统资源、服务或功能）。

![/images/2021-03-23/Untitled.png](/images/2021-03-23/Untitled.png)

其中 `ipc_kobject_t kobject` 这个成员是个task对象（可以理解为进程），task对象中包含了其虚拟地址空间 `vm_map_t map`、mach ports命名空间、线程空间以及进程信息 `bsd_info`等。

![/images/2021-03-23/Untitled%201.png](/images/2021-03-23/Untitled%201.png)

***Potential chain: mach_port  —> ipc_port —>task_port —> kernel task —> kernel task_port***

## 2 POP攻击归纳与建模

### 2.1 POP攻击

![/images/2021-03-23/Untitled%202.png](/images/2021-03-23/Untitled%202.png)

这类攻击之前已经在iOS jailbreak中使用了，本文则是先对这类攻击进行了系统的归纳。首先，其条件是存在能用于破坏内核对象的内存破坏漏洞，然后借助图3中的这几个攻击原语一步步提升：

- 起点：通过内存破坏漏洞来在伪造一个可读可写的 `ipc_port object` ，并获取对应的send right。

    常规情况下，在用户态只能拿到 Task port 的句柄，若要拿到地址，需要通过某种方式迫使内核分配 Task port 的指针到我们可读的内核区域（UAF）。

    例如，通过 OOL(out of line) Message 泄露 Port Addr，用Heap Spraying 将 ipc_port pointer 数组分配到UAF已释放区域。OOL message的消息体包含接收者虚拟地址空间以外的内容，而且如果***out-of-line的是 mach_port句柄，在 copy 时会将其转换为句柄对应的 ipc_port 的地址***。

![/images/2021-03-23/Untitled%203.png](/images/2021-03-23/Untitled%203.png)

- Querying Primitive (QP)：用于打破基于随机化的保护策略，例如random_free_to_zone引入的freelist的随机化。借助QP原语（需要多重），能够借助错误返回值等有限的信息来找到corrupted port对应的handler。

    ![/images/2021-03-23/Untitled%204.png](/images/2021-03-23/Untitled%204.png)

    暴力猜测global kernel object “Mach Clock“的地址，从而bypass KASLR

- Kernel Memory Read Primitive (RP)：用于获取内核内存读取能力。 该原语基于对类型完整性的不充分检查来滥用破坏的端口对象。

    RP通过一些system call 在内核空间和用户空间直接拷贝敏感数据，例如 `pid_for_task()` 不检查传入的task的有效性，而是直接调用 `get_bsdtask_info()` 和 `proc_pid()` 将 `task->bsd_info->p_pid` 复制到用户空间。于是可以通过伪造 `IKOT_TASK Object`  ，调用这个函数来获取任意内核内存读的能力。

    ![/images/2021-03-23/Untitled%205.png](/images/2021-03-23/Untitled%205.png)

- Kernel Memory Write Primitive (WP)：依赖RP获得的内核信息，WP原语能够利用伪造的ipc_port对象来修改内核内存。

    基于当前进程的 task_port 可以枚举出所有进程，在这个过程中需要数百次的 Kernel Read。由于 proc 是一个双向链表，可以从当前进程开始向前枚举，直至 pid=0，再从 kernel task 中取出 `vm_map`。将 `vm_map` 写入fake port，就能得到一个合法的 `kernel task port` 。【到此得到tfp0，后面的步骤继续提升权限】

- Privileged Purpose Primitives (PPP)：XNU向特权端口提供了一些强力的系统调用，PPP原语就是获取权限向这些特权端口发送消息来得到这些强力的系统调用。

    例如通过RP原语去创建 fake kernel task 来绕过 ownership check从而调用 `mach_vm_*()` 来篡改内核内存。

- 其他攻击原语：作者文章中讨论了任意代码执行原语ACEP。与之前的QP，RP，WP和PPP不同，ACEP着眼于破坏函数指针。

    例如IOKit 为 userland 提供了 IOConnectTrapX 函数来触发注册到 IOUserClient 的 IOTrap。该函数对应内核中的 `iokit_user_client_trap` 函数，会首先将IOUserClient 句柄转换为内核对象，随后从 userClient 上取出 IOTrap 执行对应的函数指针，就可以篡改该函数指针。

    ![/images/2021-03-23/Untitled%206.png](/images/2021-03-23/Untitled%206.png)

借助这五种原语，就能构建POP攻击链。作者文中以用于iOS 10.2越狱的Yalu Exploit为例子进行了POP攻击的具体说明。

### 2.2 POP原语建模

进一步地，作者尝试归纳原语搜索的方法论，于是为前面的五种原语分别归纳了模型（可以理解为label、pattern、object、Entry with object的四元组）

- *每个原语的模型*

    ![/images/2021-03-23/Untitled%207.png](/images/2021-03-23/Untitled%207.png)

    ![/images/2021-03-23/Untitled%208.png](/images/2021-03-23/Untitled%208.png)

    ![/images/2021-03-23/Untitled%209.png](/images/2021-03-23/Untitled%209.png)

    ![/images/2021-03-23/Untitled%2010.png](/images/2021-03-23/Untitled%2010.png)

    ![/images/2021-03-23/Untitled%2011.png](/images/2021-03-23/Untitled%2011.png)

### 2.3 POP searcher

基于XNU源码和上述模型：

- 获取所以的mach system call entry和mach port object

    ![/images/2021-03-23/Untitled%2012.png](/images/2021-03-23/Untitled%2012.png)

- 生成AST，并为每个entry生成Mach port-oriented control flow graph（POC）
    - 三种节点：statements (Stmt), expressions (Expr), and declarations (Decl)
    - 根据共38种Mach port objects来迭代生成相关的POC
- 根据原语的模型来匹配

    结果如下：

    - QP：在Mach子系统中找到了436个查询原语，且一些能绕过zone elements randomization。
    - RP/WP：共发现29个原语，大多在TASK和 MACH_TRAP子系统中。
    - PPP：在HOST、PROCESSOR 和 TASK子系统中共发现176个PPP原语。
    - ACEP：共72个，大多在DEVICE子系统，TIME中也有一些。

## 3 PUSH防御设计

总结了POP后，作者提出了Port Ultra-SHield (PUSH) 作为防护，如图7所示。由三部分组成：

- POP primitive searcher：根据之前归纳的模型从XNU源码与kernel cache中查找潜在的POP原语；
- PUSH policy generator：为Mach port对象采用对应策略从而在特定的Mach系统调用前后保护其完整性，例如坚持对象的地址或数据。

    ![/images/2021-03-23/Untitled%2013.png](/images/2021-03-23/Untitled%2013.png)

    - 其中*mpc_ops是callback函数，对kernel object各个阶段的各种event进行检查
        - Kernel object address examiner
        - Kernel object querying examiner
        - Kernel task examiner
        - Kernel object data examiner
- PUSH checker：用于部署PUSH policy generator，相当于kernel extension。

    ![/images/2021-03-23/Untitled%2014.png](/images/2021-03-23/Untitled%2014.png)

![/images/2021-03-23/Untitled%2015.png](/images/2021-03-23/Untitled%2015.png)

## 4 评估

在评估中，作者对表2的一系列漏洞进行了测试，涵盖了macOS 10.12-10.15和iOS 10-13。

![/images/2021-03-23/Untitled%2016.png](/images/2021-03-23/Untitled%2016.png)

其表现用插桩点IPs来表征，作者对IP的数量和具体内容做了分析，并通过case study进行了具体说明。除此之外，也对性能开销进行了测试，只有约2%的额外开销。测试的设备数量相当多，在阿里的40000多台macOS设备上部署了PUSH，结果显示确实能有效防御公开的18个exploit和一个0day exploit，也得到了苹果的致谢。

- 具体图表结果

    ![/images/2021-03-23/Untitled%2017.png](/images/2021-03-23/Untitled%2017.png)

    PUSH A (without kernel object address examiner) and PUSH B (with kernel object address examiner). Benchmark: LMBench [29], wrk HTTP benchmarks [44] for Apache, and Sysbench benchmarks [40] for MySQL

    ![/images/2021-03-23/Untitled%2018.png](/images/2021-03-23/Untitled%2018.png)

    ![/images/2021-03-23/Untitled%2019.png](/images/2021-03-23/Untitled%2019.png)

    ![/images/2021-03-23/Untitled%2020.png](/images/2021-03-23/Untitled%2020.png)

![/images/2021-03-23/Untitled%2021.png](/images/2021-03-23/Untitled%2021.png)

## 5 总结

最后总结几句：这篇文章的研究思路是比较讨巧的，既有攻击也有防御，其攻击（POP）并非首创，而是归纳了过去的一些exploit，比较系统地总结了一套方法论与模型，然后对应这个方法论进行了防御（PUSH）的设计，这种防御的论证和实验也是难点。这类文章的内容非常充实，很值得去精读学习，包括其引用的文献资料，都可以作为进一步学习的材料。当然纸上得来终觉浅，还是得硬着头皮去实操，去积累原语。