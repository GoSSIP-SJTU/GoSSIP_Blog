---
layout: post
title: "BIGMAC: Fine-Grained Policy Analysis of Android Firmware"
date: 2020-09-29 14:39:12 +0800
comments: true
categories: 
---

> 作者：Grant Hernandez1, Dave (Jing) Tian2, Anurag Swarnim Yadav1, Byron J. Williams1, and Kevin R. B. Butler1
> 
> 单位：1Florida Institute for Cyber Security (FICS) Research, University of Florida；2Purdue University, West Lafayette, IN, USA
> 
> 会议：USENIX Security‘ 20
> 
> 链接：https://www.usenix.org/system/files/sec20-hernandez.pdf

## 简介

作者开发了一个细粒度的访问控制分析的框架，静态提取Android固件的信息，构建运行中系统的安全状态。作者在Samsung和Pixel的四个版本的Android固件上运行，结果可以恢复到高达74.7%的进程凭证，并且对文件系统的DAC和MAC信息恢复的准确性超过98%。

作者对Samsung S8 +和LG G7固件的评估揭示了多个策略问题，包括LG上的不受信任的应用程序能够与内核监视服务进行通信，Samsung S8+允许从不受信任的应用程序到某些root进程的IPC，至少有24个具有CAP_SYS_ADMIN capability的进程。

<!-- more -->

## 背景

### Linux的访问控制

Linux的自主访问控制（DAC）包括给进程和文件对象创建和赋值UID和GID和他们对应的读，写以及执行的访问权限。有root权限的用户（UID=0）可以覆写DAC的权限。

为了防止root进程被攻破，Linux内核设计了Capability-like (CAP) 系统，将特权分割成了38种不同的capability。

Linux的强制访问控制（MAC）最广泛的实现为SELinux。SELinux有三个核心的部分：主体，对象和动作。主体是负责对象之间的信息流交互的进程或者设备，在allow规则下可以读取或者修改对象，从而导致系统状态的改变。对象是文件，socket和网络接口等资源。每个对象都有一组权限和一个类标识符，用于定义其目的和对象处理的服务。 这些allow规则形成安全策略，理想情况下，该安全策略授予完成任务所需的最小权限集。 主体和对象都有一组安全属性，操作系统可以查询这些安全属性以确定策略是否允许请求的操作。

### Android的安全模型

**Android的DAC**

系统进程有固定的UID/GID，动态安装的应用的UID在一个范围内动态生成。高权限的应用一般以`system`（UID 1000）用户运行，或者使用一个特定角色的UID，比如`radio`，`graphics`.

**SEAndroid**

SEAndroid的策略基于user，role，type和level的一组标签制定规则。在Android上，type是最基础的标识符，策略决定了一个进程可以访问什么样的type和action。SEAndroid还拓展了一些服务和属性的访问类。

SEAndroid用户空间对现有SELinux组件进行了修改，包括：init，Zygote，bionic（Android C库）和package manager。 Android init进程负责在启动过程中尽早加载安全策略，执行init.rc里的命令，并强制执行安全策略（即访问系统属性）。Zygote进程负责产生Android应用程序进程。 Zygote在系统启动时启动并加载通用framework代码。然后Zygote可以设置socket接口的安全标签和运行应用程序的安全上下文。bionic获取和设置文件系统属性并存储文件安全标签。package manager对应用程序请求的权限进行决策，以确定是否可以实际授予所请求的权限。这些用户空间组件在内核层之外实施SEAndroid安全策略。

## Design


![](/images/2020-09-29/upload_7c12466010eeb24cb88781312a5d913c.png)



### 安全策略提取

**初始化启动仿真**

为了获取/data,/dev,/sys这几个分区的策略信息，需要做设备启动的仿真，因为这几个分区不在静态的固件中，而是启动的时候动态创建的。

Android的init system有两个组件，`init`守护进程和`uevent`守护进程。`init`守护进程负责启动和管理native守护进程服务，`uevent`守护进程用于监视内核的设备状态变化提醒。init是第一个运行在系统上的进程，处理开机状态变化，管理服务，执行RC (Run Command) 。BIGMAC实现了影响安全状态的关键RC，以在继续进行图形实例化之前捕获目录，文件的创建以及文件所有者，组和权限模式的更改。 此外，作者保存服务定义，以供以后生成流程和模拟其运行时凭据。

在启动过程的早期，init会生成`uevent`守护程序。这基于简化的配置文件为/dev和/sys目录中的某些文件设置了定制的DAC信息。`ueventd.rc`格式是每行一个DAC条目。

![](/images/2020-09-29/upload_a97fe9ef6d4a795c66657f41c5cf4ff3.png)


**支持文件恢复**

![](/images/2020-09-29/upload_ac1315623f3d18d813aaa92205650954.png)


为了将抽象的，仅MAC的SELinux类型完全实例化为进程和对象，需要将它们与包含DAC和capability信息的具体文件相关联。

作者反编译二进制的SEPolicy获得访问向量规则（AVRules）,该规则通过**动作和类**链接**源和目标type**。通过AVRules生成主体图$G_s$，$G_s$是所有派生图的起源（如图3所示）。该策略包括SELinux使用的所有类型T和属性A。从这个抽象策略中，所有类型被划分为主体（域）S和对象O，并通过将它们的策略类型与文件系统（支持文件）上的实际文件F相关联来实例化它们。因为对于文件对象来说，它们的类型直接链接到提取期间捕获的文件系统上具体文件的类型。为此，必须通过对象执行来得到与流程转换相关的Type Enforcement规则（TERule）。

TERule连接两个domain $S_i$ 和$S_j$通过对象$O_j$和类$C_p$的二元组：$S_i -(O_j,C_p)-> S_j$。这被视为域$S_i$使用对象$O_j$（实例为$F_j$）经由类$C_p$过渡到$S_j$的域。这是binary $F_j$执行期间进程转换的显式编码。通过解析这些规则，可以找到与subject类型关联的二进制文件。作者将具有至少一支持文件的主体集定义为SB。主体可能有多个支持文件，如果没有，则无法完全实例化。 SEPolicy中允许init执行mediaserver的示例规则为：

```
allow init mediaserver_exec:file {open, read, execute};
allow init mediaserver:process {transition}; 
type_transition init mediaserver_exec:process mediaserver;
```

其中，$S_i = init, S_j = mediaserver, O_j = mediaserver\_exec, C_p=process$

并非所有Android域都具有显式的TERule，还有一种通过`process`类进行动态转换的AVRule。除了没有backing object外，它与TERule相同，这意味着主体无需`exec`即可更改其MAC标签。执行这些转换的常见域是init和Zygote（Android上）。 Zygote在启动时由init启动，并fork应用进程，根据应用程序的类别（不可信，系统，特权等），新的进程将动态转换到新域。应用程序由用户动态安装，因此不能具有硬编码的TERule。除非由seapp_contexts自定义外，某个类的所有应用程序共享相同的安全标签。通过恢复这些动态转换边，可以重新创建整个主体类型层次结构，以供以后在恢复实例化的进程树时使用。

### 数据流图

现在有了完整的主体节点S和有具体支持文件的节点SB，作者从主体图$G_s$创建数据流图$G_d$。在原始SEPolicy中，每个AVRule都包含单独的多个访问向量，例如读取，写入，打开，getattr等，但作者只对读写感兴趣，只保留了读和写的操作如表8所示。

![](/images/2020-09-29/upload_1b5ead8ecd7bb3e9e8e663313e8c3d39.png)


由于作者感兴趣的是实例化图，因此不在SB中的S作者不关注。如果没有至少一个关联的可执行文件，则该主体被认为是抽象的。因此，对于每个SB，作者会检查到其他S或O节点的所有AVRule，忽略subject-subject的边。

模型将对象分为两组：IPC对象$O_{IPC}$和文件对象$O_{File}$。对象之间的主要区别在于IPC有一个单独对应的主体（创建者/管理者），并继承其凭据，而文件对象包含一个或多个支持文件$F_i$，每个文件都有单独的MAC/DAC/CAP数据。这种拆分使得作者能够根据对象的用法和上下文有选择地实例化对象。

Android定义了用于中间件的特定AV类，包括`property_service`和 `service_manager`。作者忽略了property service(在Android下是init进程），因为不太可能允许从很多进程init可执行路径。对于service_manager，作者检查具有`add` AV的AVRule，这使得能够找到它属于的域。

从AVRule边推断出对象后，作者将这条边插入到$G_d$中。

作者对属性$A_s$进行了扩展，没有直接将所有这些属性链接到图中，而是将它们的所有边完全扩展到了每个成员中。虽然增加了边数，但是在图查询期间不再需要考虑属性的成员。该图是二分图，在最坏的情况下，具有 $O(|S|*|O|)$ 个边和$O(|S|+|O|)$个节点。

### 进程膨胀

恢复了主体层次结构和支持文件后，作者使用虚拟PID，将主体节点完全实例化为单个进程实例。从root `kernel`主体开始，fork `init`主体并分配root的起始凭据（uid=0, gid=0, groups=[], sid=u:r:init:s0, cap=ALL）。内核分配的PID为零，而PID的初始值为1。

作者使用之前提取的服务定义检查和过滤所有init子进程。对于init的每个子主体以及该主体的每个支持文件，进行查找与虚拟文件系统中的支持文件路径匹配的服务定义（已启用，不是oneshot进程）。在潜在匹配的情况下，给服务分配定义的用户，组，补充组，凭据和安全性标识符。如果定义中不包含这些值，则被视为具有所有capability的root权限服务。

为进程分配凭据后，它会以适当的Linux进程语义（父，子，继承的凭据等）插入到具体的进程树$G_p$中。模拟了启动过程后，作者利用Android域的知识来修复Zygote fork的进程。这些包括应用程序和system_server，其中system_server的凭据硬编码在Zygote源代码中。

### 实例化攻击图

作者将所有文件节点拆分为单独的文件实例，构造新图$G_f$。例如，system_data_file对象具有许多支持文件。为了确保考虑到每个文件的DAC信息，作者将有多条边的节点进行了拆分，这大大增加了边的数量，但可以为DAC和MAC查询提供具体的true或者false答案。

使用数据流图$G_f$和进程树$G_p$生成$G_a$。对于$G_p$中主体$S_j$的每个进程$P_i$，将$G_f$中$S_j$的所有出入边复制到进程树中的$P_i$中，以查询允许原始MAC策略读取或写入的具体对象。

### 基于逻辑的查询引擎

```
query_mac(S,T,C,P)
query_mac_dac(S,T,C,P)
query_mac_dac_cap(S,T,C,B,P)
query_mac_dac_cap_ext(S,T,C,B,E,P).
```

$S$ 为起始节点，$T$ 为目标节点，$C$ 是限制路径长度的参数，$B$ 是目标Linux capability；$E$ 决定外部攻击界面的类型，比如USB或者蓝牙；$P$ 包含返回的路径。

起始节点和目标节点都可以是一个进程或者一个对象（比如，文件或者IPC），也可以是通配符（在Prolog中用下划线表示）。每个查询接口应用不同的策略层和过滤机制。比如，`query_mac_dac_cap_ext(_,zygote,3,CAP_SYS_ADMIN,usb,P)` 代表请求所有针对Zygote进程的（攻击）路径，通过MAC和DAC检查，最大路径长度为3，有CAP_SYS_ADMIN capability，并且可以从USB连接启动。由于攻击图是基于MAC策略构建的，因此MAC允许图中的每个可行路径。`query_mac_dac_cap_ext`在图中寻找一个条路径：

```
find_a_path(S,T,C,B,E,P) :-
	graph_travel(S,T,[S],P,C), 
	dac_path(P), 
	cap_path(P,B), 
	ext_path(P,E).
```

`graph_travel` 使用DFS在基于MAC策略的图上寻找路径。

`dac_path`谓词通过使用`dac`谓词查看路径中的每个相邻节点对来检查相应的DAC策略。每对节点`(A,B)`是进程和系统对象的组合。`dac`谓词检查一个节点root user，group和其他基于DAC的信息。

```
dac(A,B) :dac_sub_obj(A,B); dac_obj_sub(A,B).
dac_sub_obj(A,B) :is_sub(A), dac_sub_obj_allow(A,B).
dac_obj_sub(A,B) :is_obj(A), dac_obj_sub_allow(A,B).
dac_sub_obj_allow(A,B) : is_root(A); is_owner(A,B); group_sub_obj_allow(A,B); other_sub_obj_allow(A,B).
dac_obj_sub_allow(A,B) :is_root(B); is_owner(B,A); group_obj_sub_allow(A,B); other_obj_sub_allow(A,B).
```

`cap_path`谓词通过检查路径中的最后一个进程节点来检查给定路径是否可以有某种Linux capability。也就是说， 如果最后一个节点是系统对象，还需要查看前一个节点。 因为每个进程节点都对其capability信息进行了编码，所以最后的检查是查看请求的capability是否包含在该节点的capability列表中。

```
cap_path(P,C) :cap_last(P,C); cap_prev(P,C).
cap_last(P,C) :last(P,A), is_sub(A), cap_supp(A,C).
cap_prev(P,C) :prev(P,A), is_sub(A), cap_supp(A,C).
```

类似地，`ext_path`谓词通过检查路径中的起始节点来检查给定路径是否从外部攻击面（例如USB）开始。 如果起始节点是系统对象，并且可以通过外部连接访问，则该路径是可以从外部触发的攻击路径。 因为每个系统对象都对其外部连接信息进行了编码，所以最终检查还是特定外部攻击与节点的攻击面列表之间的成员检查。

```
ext_path(P,E) :first(P,A), is_obj(A), ext_supp(A,E).
```

## 实现

BIGMAC基于Python3.6和SWI-Prolog，并且部署了NetworkX库用于操作图，SETools包反编译SEPolicy得到原始的type和rule。

### 固件提取

作者基于之前的工作实现了提取固件的工具，可以提取Google，三星和LG映像。

提取工具首先以递归方式解压缩固件映像，然后根据文件类型处理每一层。对文件系统进行遍历，使用`stat`，`getxattr`和`readlink`获取所有元数据，并将其存储在按文件名索引的字典中。接下来，使用正则表达式遍历内存虚拟文件系统（VFS），以提取出图1中所示的每个文件。此外，提取出build.prop文件，以解析包含关键元数据的属性，例如Android版本和硬件配置。将所有这些原始文件保存在image-specific数据库中，以供以后处理。

![](/images/2020-09-29/upload_090843a59f157904c3f0ac24c5d07fac.png)


### 系统启动模拟

为了恢复运行的系统的大致状态，在各个阶段组合了所有已知的策略文件。除了原始SEPolicy之外，最重要的文件集是Android初始化脚本。这些是基于文本的，类似于shell的命令，它们按块顺序执行。

在BIGMAC中，作者实现了`mkdir`，`chown`，`chmod`，`trigger`，`enable`和`mount`命令。前三个更改文件系统的状态，包括DAC信息。`trigger`引发一个事件，该事件导致其他部分被执行；`enable`启动服务；`mount`挂载文件系统。 `mount`命令的处理特别重要，因为它会影响在模拟`restorecon`过程中分配文件的有效SEContext。

大多数错误的MAC/DAC数据来自未处理供应商和设备型号在init系统中一些自定义的过程。例如，在Pixel 8.1.0映像上，必须将属性`vold.decrypt`设置为`trigger_post_fs_data`，以便初始化模拟器执行创建/data目录的正确引导部分。特别地，某些OEM添加了自己的自定义组和用户Android ID（AID），这些ID与Android平台有所不同。作者计划从Bionic libc.so二进制文件中提取导出表和android_id。

### Android 凭证模拟

引导init并修改文件系统后，作者使用从init文件中恢复的服务定义将运行时凭据正确分配给实例化进程。 服务定义包含其初始用户，组和功能。这些有助于获取准确的DAC和CAP信息，并提高攻击图的正确性。

## Evaluation

###  Ground Truth 对比

作者对root过的Google Pixel 1和Samsung Galaxy S7 edge里MAC，DAC和处理信息进行对比静态分析的准确性。在Pixel上，作者分别刷了三个不同版本的AOSP：Android 7.1.2, 8.1.0 和9.0.0。

**文件权限**

如表1所示，作者能够从静态固件（包括/vendor和/system）完全复制正在运行的系统中的大多数主要目录。

“Extra Files”（误报）行所示/dev的误报很高，是由于解析`uevent.rc`文件来推断潜在的设备节点比正在运行的手机上实际包含的设备节点更多（没有解释）。然而，在不恢复/dev的情况下，许多SELinux上下文具有没有支持文件，这意味着无法完全实例化关联的文件对象，从而导致漏报。

一些目录（即/data和/odm）具有较高的漏报。原因是，这些文件系统仅在首次引导后创建，或者是特定于供应商的。例如，/data中的Pixel 1（8.1.0）上缺少的5,350个文件大部分是应用程序的缓存。不过，本文攻击图模型主要集中在系统级目录和文件上，所以可以放心地忽略这些目录的详细内容。

在可以恢复的文件DAC和MAC数据中，所有映像的TP均大于98％。

![](/images/2020-09-29/upload_8469fab72debef38195d9cf0d3e8536b.png)


**进程树**

进程恢复的结果如图4所示。对于1a列中的S7 Edge，作者实例化了49个进程，其中25个对于运行中的设备而言是完全准确的，其中20个是部分准确的，还有4个是额外的（不在真实设备上运行）。在1b列中缺少的进程中，有55.5％是应用程序进程，由于作者重点在本机守护程序上，因此未实例化应用程序进程；20个进程是本机守护程序，由于各种原因而未实例化，例如已实例化的进程创建了子进程。在Pixel 1 9.0.0上表现最好，其中74.7％的进程有匹配的进程，而只有9个缺少的本地进程。总体上看，超过50％的进程完全匹配了实际进程。

![](/images/2020-09-29/upload_7982475ad88a745f33f5f2b266517324.png)


### 攻击图查询

作者从Samsung S8+ 8.0和LG G7 9.0固件上生成攻击图。

**路径缩减**

如表2，作者进行了只有MAC策略的查询`query_mac_only`和对MAC+DAC策略的查询`query`。`vold`进程负责安装和管理磁盘卷。在S8+上MAC-only的查询长度为4的查询仅有100K，而考虑DAC后仅有14K个路径。

![](/images/2020-09-29/upload_62ca16d615977316381e24d962d425e9.png)


如图5的三个目录或者文件，单独考虑MAC策略是可写的，但是考虑DAC后，由于untrusted_app进程不是`media_rw`组成员，所以不可以访问这个目录。可能在运行时应用程序进程获取到group或者DAC权限发生改变。 BIGMAC当前无法使用安全策略状态的快照。应用DAC信息虽然增加了查询运行时间，但它提供了实际系统中MAC和DAC策略所允许的实际路径结果。先前工作显示的纯MAC路径具有更多的误报，导致结果不够准确，无法从中得出安全性结论。

![](/images/2020-09-29/upload_8a235b482adf491f082f6fd890cc7887.png)


**特权提升分析**

CVE-2018-9488：zygote进程crash后，执行`crash_dump64`到`crash_dump`域，在该域可以通过ptrace系统调用读写`vold`的内存（MAC策略允许）。

![](/images/2020-09-29/upload_a74bd425acd437f51256aeecf78a3d3a.png)

BIGMAC 使用查询`query(process:zygote,process:vold,4)`发现了这条路径。这个查询返回700+条路径，所有都包括文件，只有一个包括域转换到crash_dump。

进一步，作者通过`query(_,transition:crash_dump,1,CAP_SYS_ADMIN)`发现了24个其他的守护进程可以通过相似的路径进行特权提升，比如说用于解析APK文件格式的`installd`。



**进程攻击界面**

表4是作者查询了限制路径长度为1和2的时候的所有可能路径。

![](/images/2020-09-29/upload_5da14b4afc599a10eb1ae34d0050be5b.png)


作者在LG上发现了11个`unstrusted_app`可达的IPC，其中一个如图7所示可以和内核监控服务进行连接。这个服务允许应用从内核获取完整性检验的信息。对服务的Binder proxy接口的进一步调查表明，存在仅限system-app的中间件检查。但如果有个proxy没有进行检查则会暴露这个服务，所以作者认为这个MAC策略应更改为仅允许system_app使用而非untrusted_app类型。

![](/images/2020-09-29/upload_c2e03d07b90ecb54fa187d2515a35465.png)


**寻找capability**

作者对两个非常危险的capability `CAP_DAC_OVERRIDE`和`CAP_SYS_ADMIN`进行了查询:

```
query(untrusted_app,_,2,CAP_SYS_MODULE)
query(untrusted_app,_,2,CAP_DAC_OVERRIDE)
```

LG G7上，一个应用攻破`netd`和`zygote`（root）后可以通过binder覆盖DAC。对于SYS_MODULE，它必须通过binder到`system_server`。

Samsung S8+除了上面的路径外，还有三条OEM特有的路径到进程`hal_iop_default`, `hal_perf_default`, `healthd`（都是root）。

 此外，在默认的AOSP SELinux策略中，不允许任何进程通过binder请求与untrusted_app域通信。

**外部攻击界面**

AT分发程序主要负责将AT命令分发到不同的本机守护程序和应用程序。如表5是Samsung S8+上可以访问并且连接到外部设备的路径，包括/dev/mtp/usb, /dev/block/platform/soc/7464900.sdhci, /dev/mdm。但这个地方作者并没有对路径进行分析。

![](/images/2020-09-29/upload_1bc5377551e2183240a1d010771d84a0.png)

