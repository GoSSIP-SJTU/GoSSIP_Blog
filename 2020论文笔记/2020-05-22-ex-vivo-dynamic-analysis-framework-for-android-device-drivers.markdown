---
layout: post
title: "Ex-vivo dynamic analysis framework for Android device drivers"
date: 2020-05-22 15:14:28 +0800
comments: true
categories: 
---

> 作者：Ivan Pustogarov，Qian Wu，David Lie
>
> 单位：University of Toronto
>
> 会议：S&P'20
>
> 链接：[Ex-vivo dynamic analysis framework for Android device drivers](https://www.computer.org/csdl/proceedings-article/sp/2020/349700b434/1j2Lgnwcu9q)

## 简介

在设备上进行在线分析，会需要特殊的权限，并且对于不同设备机型的分析的要求不同，难以实现大规模分析。

另一种方法是离线分析，通过仿真硬件环境实现分析。但是其中存在两个挑战：1）驱动有一些专有硬件依赖和设备特有的硬件组件；2）驱动和宿主kernel之间的软件依赖。

作者开发了一个框架evasion可以规避硬件和内核的依赖初始化驱动，并且依赖这个框架开发了一个动态调试离线设备的框架EASIER (Ex-vivo AnalySIs framEwoRk)，分析IOCTL的权限提升漏洞。

作者收集3个不同的Android内核和内核的子系统的72个平台设备驱动，evasion能够成功加载77%的设备驱动。使用26个已知漏洞测试框架EASIER，可以触发其中81%。使用EASIER去fuzz驱动，发现了29个漏洞。作者人工分析后在小米的Android内核中发现了12个zero-day，其余17个来自MSM内核，但是是在老版本中发现的。

<!-- more -->

# 背景

### 可加载内核模块

一些设备驱动可以被编译为可加载的内核模块，加载后和其他内核模块具有相同的权限。

### platform bus和device tree文件

当今的Android智能手机将大多数组件/外围设备集成到一块板上。这样的集成外设可以是SoC的一部分，也可以使用I2C或AMBA总线，它们都不支持设备发现（与PCI或USB设备总线相反）。 Linux内核将虚拟“平台总线”用于这些集成的外围设备，并且系统开发人员使用device tree文件管理外围设备。

![img](/images/2020-05-22/listing1.png)

device tree文件描述了特定电路板的硬件配置，并为每个外设提供了纯文本描述。例如，listing1显示了描述高通Snapdragon 200 SoC的Camera ISP（图像信号处理器）的条目之一。包含内核/驱动程序在初始化期间需要的属性，例如MMIO内存范围（字段reg）和中断号（字段interrupts）。包含该条目的已编译device tree文件会在引导时传递给内核（例如，在uBoot的情况下使用devicetree_image选项），使内核知道板上有哪些硬件设备。device tree文件中的每个外围设备均由其compatible属性来标识。

device tree文件是将设备信息传递到内核的主要方法，还有一种较旧的选择是broad文件，它以纯“ C”形式描述设备并直接编译到内核中。board file已过时，仅由较早的内核使用。

### 模块加载和初始化

当模块加载到内核中时，它首先经历执行符号重定位的链接过程：模块可以在运行时从主机内核调用函数，但是在模块编译期间这些函数的位置未知。因此，在将模块加载到内核后，内核会使用模块的重定位表将这些函数调用链接到其实现。如果内核缺少该模块所需的任何函数，则中止模块加载。

![img](/images/2020-05-22/fig1.png)

完成重定位后，内核将调用模块中的几个函数以完成驱动程序和外设初始化，如图1所示。每个模块都定义一个init_module函数和一个可选的Probe函数。一旦完成重定位，内核就会执行init_module函数。通常，在平台设备的情况下，此函数向外围总线之一注册驱动程序，并提供指向其probe函数的指针，例如通过使用platform_driver_register API。然后，与总线相关的代码将遍历现有设备的列表（从device tree文件中预填充），并尝试查找匹配的设备。确切的匹配过程跟总线有关，对于平台总线，它基于以下条件来匹配驱动程序和设备：a）device tree文件中的compatible属性； b）设备名称；或c）驱动ID。如果找到匹配的设备，则总线代码将调用驱动的probe函数。 probe函数通常在/dev/文件夹中创建一个新文件，并使用register_chrdev()注册IOCTL /read/write system call handler。然后在设备文件上进行的系统调用由注册的handler处理。 

### IOCTL system call

 IOCTL系统调用提供了比标准read/write系统调用更多的功能，允许用户空间发布不同的命令给驱动，可以传递和接受C结构的任意消息。IOCTL包含一个“命令”字段，可以被设置成一系列与设备有关的值，以及一个"参数"指针，指向一个任意的C结构的外设。

### 内核-用户空间安全数据拷贝

驱动和用户空间的程序进行数据交换，通过copy_from_user()/copy_to_user() 函数拷贝数据。

## Evasion kernel

![img](/images/2020-05-22/listing2.png)

第5行和第9行，如果是空值就直接会结束；也不能单纯设置为非空，第12行会使用这个值，如果返回结果不对也会结束。

要求：1）满足驱动的软件依赖；2）隐藏硬件依赖；3）确保kernel和驱动之间传递的数据结构格式的一致性。

evasion kernel基于 Vanilla Linux kernel ， arm32的vexpress开发板，arm64的virt开发板。

### 软件依赖

Evasion kernel的目的不是完全复制原kernel中的函数功能，只是为了能够正常加载驱动（无crash）。

作者实现了两个stub去替代丢失的函数。一个是返回0值的**stub0**，替代所有返回值是int的函数，因为Linux kernel里的函数经常返回0表示成功，返回其他非零值为错误代码。另一个是开辟一块内存空间的**stubP**，用于替代返回值为指针的函数，这种情况下原函数一般是返回一个指向结构的指针。

为了推断缺少的函数返回类型，evasion kernel从模块的二进制文件中获取模块导出的函数的列表，并在主机内核源代码中检查其签名。 它将此信息作为新的ELF section作为二进制模块本身的一部分附加。 然后，evasion kernel的自定义模块加载子系统使用此信息来决定选择哪个功能stub。

也有些函数会使用1作为成功的返回值，所以作者也实现了**stub1**，在模块加载失败后尝试使用stub1代替stub0。72个驱动中只有2个是这种情况。

### 硬件依赖

因为不同总线的硬件依赖的处理方式不同，作者这里只关注使用最多的platform总线。

**1）重用device tree 条目**

如果主机内核device tree文件可用，作者直接在evasion kernel中进行使用。但是device tree条目中存在对其他硬件的交叉引用的情况，比如引用不同的中断处理程序和时钟。evasion会先将其替换成已有的中断处理和时钟。为了找到匹配的条目，evasion首先在不消除硬件的情况下加载驱动，然后观察驱动提供的compatible属性，再去主机kernel中搜索对应的条目，加载到现有的device tree中。

如果主机内核的device tree文件不可用，或者没有搜索到对应的条目，evasion直接使用最常用的合理值生成一个条目。

**2）board file**

作者手动提取相关的board文件加入evasion。因为这种情况较少，作者认为没必要自动。

**3）忽略驱动-设备通信**

比如，在初始化期间，大多数驱动程序将使用device tree 条目中的值向映射对应页面的ioremap / ioremap_wc或of_iomap注册MMIO范围。 evasion拦截上述函数并将其重定向到kzalloc。在缺少外设的情况下，从该内存读取将返回零数组。 

![img](/images/2020-05-22/apA.png)

最后，evasion拦截并用自定义实现替换了15种行为依赖于外围设备的存在的现有内核函数，将它们与被替换的软件依赖函数区别开来，因为这些是在普通Linux内核中具有现有实现的函数，而软件依赖替换了仅存在于主机内核中的函数。

### 内核-驱动API结构布局

作者分析了多个驱动后，发现只有两个结构需要实现在驱动和内核之间的对齐，struct device和struct file。前者用于在驱动程序初始化/探测期间将信息从驱动程序传递到内核，而后者则用于在用户空间打开/dev文件时使用。 根据主机内核及其配置，这些结构将包含或缺少某些字段，并且如果驱动程序和规避内核布局不匹配，则内核和驱动程序将以不同的偏移量读取/写入信息（大多数情况下会crash）。

![img](/images/2020-05-22/listing3.png)

作者进行对齐的方法：

如果主机内核的源码不可访问，首先在Linux内核中收集一些函数，满足：1）参数有该结构，2）使用了结构中尽可能多的字段。作者使用的函数：struct device是i2c_new_device, device_resume和 device_initialize ，struct file是 __dentry_open。对比evasion和主机内核的binary，工具插入填充以使适当的字段对齐。

如果主机内核源码可以访问，那么可以跳过第一步，直接在evasion模块中列出所有的字段。通过尝试不同的配置选项进行重新编译evasion，使binary中相同struct各个字段的偏移相同。

### 代理模块

在大多数情况下，如果内核正确初始化，则会在内核的/dev目录中创建一个设备文件。稍后需要该设备文件才能将IOCTL发送给驱动程序。但是，部分情况下，由于缺少的依赖关系而无法创建设备文件，但是驱动程序已经足以工作。例如，如果在初始化结束时驱动程序要求外设返回特定值，而evasion返回的值无法满足要求。这种情况下，evasion提供了创建“代理”设备文件并将驱动程序的system call handler附加到设备的功能（通过使用kallsyms_lookup_name()在内存中查找handler）。这使EASIER可以从用户空间调用驱动程序的system call handler并进行分析。

## EASIER

EASIER首先驱动和内核的初始化状态到内存快照中。然后从文件读取输入并注入内存快照，在一个基于Unicorn库实现的CPU仿真器dUnicorn加载映像并执行。为了执行提取出的内存快照，一些脱离硬件无法执行的内核函数被替换成作者提供的等价操作。使用映像和dUnicorn，EASIER可以进行对IOCTL系统调用的fuzzing和符号执行，识别漏洞，并且自动生成触发bug的程序。

### 内存快照

第一次处理system call时，EASIER对整个easion kernel映像进行快照，转存所有内存页表和寄存器的值。EASIER使用Qemu作为仿真器运行evasion kernel和处理快照。

### dUnicorn

dUnicorn加载模拟的内存并恢复寄存器去执行快照，比如说会读取文件中的值赋值给system call的参数，这样一来，就可以针对不同的潜在恶意输入来测试系统调用。模拟CPU指令直到离开system call或者返回用户空间，特别是执行到 ret_fast_syscall指令。如果有指令想要访问没有映射的内存，dUnicorn会返回一个SEGFAULT事件。也就是说，dUnicorn只有当内核里驱动对于相同的输入crash时才会crash。

优点：约束不足的符号执行之类方法将所有未初始化的状态视为符号，然后尝试查找所有可以触发bug的输入和状态。 但是，对于像OS内核这样的复杂程序，符号状态可能会变大并导致路径爆炸。 而且，大状态也可能超出约束求解器范围。 而EASIER通过产生足够精确的初始化状态以进行动态分析来消除这些问题。

### 替换内核函数

作为CPU-only的仿真器，dUnicorn仅仿真ARM指令，而没有其他硬件。因此，dUnicorn需要拦截访问dUnicorn无法模仿的硬件并将其重定向。被拦截的函数分为三大类。

首先，由于CPU-only仿真中没有MMU，因此内核代码无法映射新的物理页面。在大多数情况下，这不会造成任何困难，因为slub分配器可能已经映射了驱动程序使用或将要请求的所有内存。但是，如果驱动程序确实在初始化后分配了内存，则dUnicorn会拦截并替换内存分配例程，例如kzalloc，krealloc和kfree，该调用将调用简单的自定义内存分配器，该自定义内存分配器从未使用的模拟内存中分配chunk。

其次，dUnicorn拦截了切换上下文的调用，例如_cond_resched。当前，上下文切换与成功返回用户空间的处理方式相同。

最后，dUnicorn选择性地禁用日志记录和输出功能，例如printk。通常可以安全地跳过此类日志记录功能，因为它们对分析没有任何作用，并且执行时间较长。禁止后可以将速度提高3倍。但是这种会丢失一些char数组没有正确以null结束的错误。

### IOCTL输入格式恢复

作者只针对IOCTL，因为IOCTL输入要求较为复杂，而read/write的输入只需要一个连续的数组和大小。

ioctl(int fd, long cmd, void *arg), 

fd是驱动文件/dev的文件描述符，cmd是IOCTL的命令编号，arg是指向一个任意的结构，格式被驱动定义,可能有复杂的嵌套结构。

listing4中是一个典型的cmd结构，中间包含两个指向动态分配内存的指针，长度由num_cfg，cmd_len确定。还有cmd存在嵌套结构，结构中存在指针指向另一个结构。由于情况比较复杂，所以要正确确定fuzz对象。

![img](/images/2020-05-22/listing4.png)

EASIER动态恢复IOCTL结构布局。可以将IOCTL的arg设置为任意值，并且dUnicorn在动态分配正确的内存大小并在正确的位置插入指针。作者基于观察：为了在用户空间和内核空间之间复制数据，驱动程序必须使用copy_from_user()内核API调用（不使用它本身就是dUnicorn可以捕获的错误）。该族中的函数，将用户空间中指向要复制的数据的地址和给出要复制的数据大小的整数作为参数。 dUnicorn截获对copy_from_user()的调用，并将其重定向到其自己的实现，该实现动态地在内存上分配所需的内存量，并为它填充随机数据并将其返回给驱动程序。这利用了以下事实：如果指针出现在copy_from_user()函数中，则驱动程序期望在该位置存在用户空间数据。

![img](/images/2020-05-22/fig2.png)

考虑一下图2中的示例。假设用户空间调用ioctl libc函数并将arg设置为0x10000000，如图2a）所示。当驱动程序第一次调用copy_from_user()时，它将使用此指针作为第一个参数，并将len1作为其期望的结构（或结构数组）的大小。 dUnicorn截获此调用，暂停仿真，并分配len1个字节的仿真内存，并为它填充随机数据，如图2b所示。 dUnicorn然后继续执行。

现在假设驱动程序期望指针指向刚复制的数据中某个（未知的）偏移量d（图2c）。它将在另一个对copy_from_user()的调用中尝试使用此偏移量的（随机）值（也存储在src2中）。 dUnicorn截获此调用并在复制的数据中搜索src2；一旦发现它给我们偏移量d。 dUnicorn然后分配大小为len2的新空间，并相应地更新src2和内存slot d处的值（见图2d）。

### Fuzzing

作者使用AFL的unicorn模式进行fuzz。AFL与其他任何用户空间程序一样执行dUnicorn。 然后，dUnicorn完成将变异的输入复制到适当的内存位置，捕获未映射的内存错误并引发SEGFAULT信号，动态拦截和重写函数调用以及动态恢复IOCTL结构的所有工作。

### 符号执行

符号执行基于Manticore框架，Manticore可以执行用户态的ELF binaries但是不能执行内核代码，作者对它进行了扩展，使它可以从内存快照中恢复执行状态。

作者使用符号执行去恢复cmd number，因为fuzzer很难去猜测这个值。作者恢复IOCTL命令号的方法基于IOCTL处理程序中具有大型switch语句的通用约定。 通常将每个switch case编译为条件分支指令。 作者为寄存器r1（包含cmd参数）分配一个符号值，并且每次执行状态在IOCTL handler内（而不是在其被调用方内部）派生时调用求解程序。

### 程序生成

每当crash发生，系统会使用触发crash的输入和恢复的IOCTL结构布局自动的生成一个C语言程序，以在真实的内核上运行触发crash，用作PoC代码并且可以检测是否为误报。

## Evaluation

### 测试已知的漏洞

作者从cvedetail.com 数据库中搜索了21个Android相关的CVE，这些CVEs属于10个不同的MSM内核（高通）的驱动，并且可以被evasion内核加载。这10个驱动的内核版本是MSM kernel 3.4/3.10/3.18，所以作者为evasion准备了相同的版本 Vanilla kernel。

后5个bug是在MSM内核新版本中被patch过的，没有申报CVE。

![img](/images/2020-05-22/tab1.png)

### evasion内核加载驱动

作者使用了62个驱动，来自MSM kernel，Xiaomi Redmi 6 kernel，Huawei P20 Pro Kernel。其中，20个驱动来自MSM kernel，作者选择了前20个包含IOCTL system call handler的驱动，并且这些驱动不在 Vanilla kernel和前一个实验中。32个来自Xiaomi（作者找到了50个不在 Vanilla kernel的驱动，但是只能成功将其中32个编译成模块，作者认为剩下18个之前版本中残留没有真正用上的）；10个来自Huawei。

![img](/images/2020-05-22/tab2.png)

失败的原因一般发生在init或者probe函数，他们需要的一些函数功能无法满足，比如初始化一个结构的字段。

只有vidc_vdec.ko和mdss_rotator.ko两个驱动使用stub1。

### Fuzzing结果

**1）测试集**

32个驱动，其中包括24个Xiaomi驱动，8个MSM内核驱动（出自之前加载测试的测试集里）。由于dUnicorn只支持32bit binary，所以无法fuzz华为的驱动（都是ARM64）。

在8核/8G内存/Ubuntu 18.04的机器上，对每个驱动的fuzz运行12小时到2天，总共运行时间715个小时。

**2）发现bug**

发现了12个小米新bug，有5个收到了确认。

17个MSM Kernel的bug但是由于作者使用了较老的版本进行分析，不清楚现在是否存在。

![img](/images/2020-05-22/tab4.png)

模糊实验还使用了一个检查器，该检查器可以检测用户空间应用程序可能导致驱动程序尝试使用kmalloc分配任意大的内存缓冲区的情况。 该检查器发现小米有13个未绑定的kmalloc使用，而MSM内核有1个未绑定的kmalloc。 

**3）误报**

MSM kernel发现了1个误报 ， 小米发现了4个。

小米误报中的3个是由于小米内核与evasion内核之间的pm_qos_request**结构定义不匹配**而导致的。经过手动分析，作者发现小米版本包含一个附加字段，导致驱动程序与evasion内核之间不兼容，从而导致错误崩溃。作者当前的evasion内核仅保证常见结构（如struct device和struct file）的布局相同，并且不能处理此结构。但是作者说可以对此进行扩展。

小米第四个误报是由于循环导致驱动程序从设备读取一个值直到获得非零值。由于每当驱动程序尝试从不存在的外围设备读取时，EASIER始终返回零，因此循环永远不会终止。但是作者认为此问题也可以归类为错误：理想情况下，内核不应仅由于外围设备故障而挂起。

MSM内核的一个误报是驱动程序未能完全初始化，但仍生成了设备文件。实际上，变量没有正确初始化，然后被驱动程序的IOCTL处理程序使用。

**4）fuzzing率和执行路径**

表III中列出了模糊测试所花费的时间，发现的代码路径数量以及每个驱动程序的模糊测试率。平均而言，EASIER使MSM内核驱动程序以每秒1167次执行模糊，而小米驱动程序以每秒525次执行模糊（在8核计算机上）。两个内核之间的差异是由于快照大小的差异所致：分别为37Mb和205Mb。与以前的hybrid系统（如Charm）相比，这种模糊处理速度提高了12个数量级，后者通过对16核机器上的驱动程序进行模糊处理，每秒可实现约20次执行。

由于缺少物理设备，因此无需重新初始化或重新启动设备（例如在崩溃之后）。在某些驱动程序上，模糊测试导致发现的路径数量很少。作者发现，通常是由于magic number而导致无法找到Fuzzer，而不是由于EASIER或evasion内核。例如，在IOCTL命令switch语句驱动程序之后，驱动程序在用户输入字段上使用了另一个switch语句。

![img](/images/2020-05-22/tab3.png)

**5) 上下文切换时停止**

如果驱动程序需要等待外围设备提供的值，则可以调用上下文切换。 dUnicorn无法模拟上下文切换，会直接停止。 这样的上下文切换可能会阻止覆盖驱动程序中的新路径。 在模糊测试期间，仅在Xiaomi驱动程序的13/267 IOCTL命令和MSM驱动程序的3/137 IOCTL命令中发生了这种暂停。

## limitation

由于缺乏实际的硬件，只能用于分析系统调用，不能分析恶意输入来自用户空间应用程序导致的漏洞。

当前的实现不支持中断，对于ARM32，仅支持平台和I2C总线；对于ARM64，仅支持平台总线。 

evasion可能会产生误报。
