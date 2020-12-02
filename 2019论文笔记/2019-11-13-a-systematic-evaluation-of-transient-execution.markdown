---
layout: post
title: "A Systematic Evaluation of Transient Execution"
date: 2019-11-13 02:12:11 -0500
comments: true
categories: 
---
> 作者：Claudio Canella, Jo Van Bulck, Michael Schwarz, Moritz Lipp, Benjamin Von Berg, Philipp Ortner, Frank Piessens, Dmitry Evtyushkin and Daniel Gruss.
>
> 单位：Graz University of Technology (格拉茨技术大学), imec-DistriNet, KU Leuven, College of William and Mary.
>
> 出处：28th USENIX Security Symposium (USENIX'19)
>   
> 原文：[paper](https://www.usenix.org/conference/usenixsecurity19/presentation/canella)

<hr/>

## 一、背景

随着时代的不断发展，计算机性能得到了不断的提升，但由于物理因素的限制，使得 CPU 的单核性能无法线性的提升，因此，增加 CPU 的核心数量以及改进 CPU 的流水线（Pipeline）成为提高 CPU 性能的关键所在。现代 CPU 的流水线并行度很高，而且 CPU 内部有很多的特性，比如后续指令的**提前执行**（Ahead of time）和**乱序执行**（Out-of-order）等。按照直觉，当某些操作对之前的指令（还没执行）出现数据依赖或控制依赖的时候，CPU 内部流水线应该会**阻塞**（Stall）。因此，为了使得流水线处于满载状态，很有必要去预测数据依赖关系以及控制流依赖等。但是，预测并不一定都是正确的，因此，当预测失误的时候，就必须把流水线清空（Flush），或者执行回滚操作（Roll-back），使得预执行的指令失效。此外，在指令遇到错误（比如：Page fault 等）的时候也需要清空流水线或者回滚。

<!--more-->
当然，还存在其它情况也需要 CPU 执行 Flush 和 Roll-back 操作，使得预执行的指令失效，以保证程序在功能上的正确性，而这些预执行的指令所处的状态就叫做**瞬态**（Transient: First they are, and then they vanish），我们把预执行过程称作**瞬态执行（Transient execution）**，而在瞬态执行过程中，就有可能会出现秘密信息泄露，然后再通过一些特殊的渠道（**Meltdown（熔毁）**、**Spectre（幽灵）** 和 **Foreshadow（预兆）** 等）就可以获得这些泄露的信息。虽然瞬态执行过程中泄露的信息在**体系结构**（ISA）上是不可见的（比如：对开发者不可见），但是在**微架构**（Microarchitecture）上是可见的（比如：这些信息会残留在 Cache 里面）。


## 二、提出的方法以及解决的问题

在这篇文章当中，作者系统化的分析了各种瞬态执行攻击和防御方法，包括 Spectre、Meltdown 和 Foreshadow。由于瞬态执行攻击的命名方式比较混乱（Confusing Naming Scheme），导致存在错误分类的情况，因此，作者纠正前人的错误，正确的给瞬态执行攻击进行分类，给出一棵**分类树**供大家参考与研究，如图 1 所示。此外，作者还**发现了 6 个未被发掘的瞬态攻击**，其中 2 个是可利用的 Meltdown 类型（在 Intel 处理器上的 **Meltdown-PK** (Protection Key Bypass) 和 在 Intel&AMD 处理器上的 **Meltdown-BND** (Bounds Check Bypass)），另外 4 个是 Spectre 类型的攻击，并在三个主流处理器产品（Intel、AMD 和 ARM）上对这些攻击进行概念验证（Proof-of-Concept）和实现。最后分析了软件中可能会产生漏洞的**代码片**（Gadget），以及这些代码片在现实世界中的盛行率（Prevalence），并给出了一些相应的防御措施。

![瞬态执行攻击的分类](/images/2019-11-13/Classification_of_transient_execution_attack.png)



## 三、技术方法

### 1. 预备知识：瞬态执行（Transient Execution）

#### (1). 指令集架构（Instruction Set Architecture，ISA）和微架构（Microarchitecture）

**指令集架构**就是软件和硬件之间的接口（也就是指令集），它定义了处理器所支持的指令、可用的寄存器和机器寻址模式等，常见的 ISA 有 x86、ARMv8 等。**微架构**是指 ISA 的实现方式，比如：指令取址译码执行访存写回、流水线的深度、CPU 内部元素（比如各种内部的 Buffer 等）之间的相互连接方式、执行单元、Cache、分支预测器等。

不管是指令集架构还是微架构，它们都是**有状态**的，指令集架构的状态包括：指令执行完之后的寄存器中的数据和内存中的数据等，而微架构的状态包括：Cache 行、[旁路转换缓冲/页表缓冲](https://blog.csdn.net/limanjihe/article/details/50174427)（Translation Lookaside Buffer, TLB）和执行单元的使用方式等。**总之，可以方便的理解为：指令集架构是开发人员可见的部分，而微架构是开发人员不可见的部分**。

#### (2). 乱序执行（Out-of-Order Execution）

现代处理器中，指令的执行一般是先把指令进行解码，然后把它分割成多个**微操作**（μOP），再执行各个微操作，这种设计方式使得处理器开发人员可以对指令的执行进行超标量优化（Superscalar Optimization），并通过一种称作**微操作更新**的方式来扩展或者修改指令的实现方式。此外，为了增强 CPU 性能，CPU 的内部通常会采用所谓的乱序执行方式设计，这使得 CPU 在执行微操作的时候，不但可以安照原来的指令流顺序执行，还可以并行的执行微操作，以此来充分利用 CPU 的执行单元提高性能。

如果一个微操作的操作数不依赖之前的数据（Available），并且 CPU 对应的执行单元处于空闲状态，那么，即使在指令流中排在该微操作前面的微操作还没有执行完，CPU 也会开始提前执行该微操作，但是，在该微操作之前的微操作还没执行完之前，该微操作的执行结果在 ISA 层面是不可见的，而 CPU 通常会使用一个所谓的**重排序缓存**（Reorder Buffer, ROB）来跟踪这些微操作的状态，并且根据前面微操作的执行情况，有序的辞退（Retire）这些微操作，并决定是否把中间状态显现给 ISA（前面微操作都正确执行）（开发者可见），或者丢弃（比如出现中断、异常等，此时需要清空 ROB）。因此，CPU 就有可能会执行一种所谓的**瞬态指令**（Transient Instruction），这种瞬态指令的执行结果永远也不会提交到 ISA 层面，既：只会在微架构层面存在。

#### (3). 预测执行（Speculative Execution）

软件实现的逻辑通常不是线性的，而是包含有众多的**分支**和**数据依赖**关系的，因此，理论上来说，CPU 在执行过程中，通常会由于分支或者数据依赖而出现**阻塞**现象（Stall），直到分支确定或数据依赖解除之后才会继续往下执行。但是，这种理想情况通常会大大降低 CPU 的性能，因此，CPU 内部就会出现各种机制来预测分支结果和数据依赖关系，使得 CPU 可以按照预测的分支继续执行，并把中间结果缓存到 ROB 中，直到依赖解除。以分支预测为例，当分支预测成功时，CPU 会把 ROB 中的预执行结果**提交**（Commit）到 ISA 层面可见的状态，否则，CPU 需要执行回滚操作返回到最后一个正确的状态，并把 ROB 清空。

#### (4). 缓存隐蔽通道（Cache Covert Channels）

Cache 是用来缓解 CPU 和 内存之间存在较大的速度差异而设计的，用来减少 CPU 访存所带来的延迟。由于 CPU 在访问 Cache 和内存的时候，它们的时间开销不一样，因此，就会出现侧信道攻击（Side-Channel Attack）和隐蔽信道攻击（Covert Channel Attack）。特别的 **Flush+Reload** 攻击技术可以让攻击者观察到跨处理器核的 Cache-line 访问，可以用来攻击密码算法、用户输入和内核地址信息等。具体而言，**Flush+Reload**  技术首先通过 **clflush** 指令来清空共享 Cache，然后等待一段时间（让受害者访问数据），再 Reload 共享数据，并检测被 Reload 的每一个 Cache-line 所使用的时间，若 Reload 所使用时间很短，则可以判定受害者刚才使用了这个 Cache-line，否则，说明该 Cache-line 未被使用过。

#### (5). 瞬态执行攻击（Transient Execution Attacks）

瞬态执行攻击是一种不在程序代码意图之内、或者不在程序数据路径上的一种**未授权的数据计算/访问**，并且它的执行结果是不会提交到 ISA 层面上，它只会在微架构上遗留执行痕迹。攻击者可以通过某些特殊的途径来部分获取/恢复这些瞬态执行结果。

![瞬态执行攻击过程](/images/2019-11-13/Overview_of_transient_execution_attack.png)

瞬态执行攻击有多种变体，但是它们的总体攻击流程基本一样，如图 2 所示，攻击者①首先通过某些手段设置好微架构的状态（例如，清空 Cache、填充（Populating）好 CPU 内部的分支预测器等）；然后②让 CPU 执行一条所谓的触发指令（Trigger Instruction），该触发指令的目的在于让后续的操作无效（文中使用 Squashed 这个词来表示），例如，一条能够产生异常的指令、或者是一个错误的分支预测指令等；在触发指令执行完之前，③CPU 将会继执行续瞬态指令序列，攻击者利用瞬态执行指令来扮演一个微架构隐蔽信道的发送端（Sender），来完成信息泄露的任务（例如：加载与秘密相关的内存到 Cache 中）；最后，在辞退（Retirement）触发指令的时候，CPU 发现是一个异常指令，或者是分支的错误预测指令，CPU 就会清空流水线，丢弃瞬态指令的执行结果，然而，在攻击的最后阶段，隐蔽信道的接收端利用某些方法（例如：通过测量特定内存块的访问时间）恢复之前未授权的瞬态指令计算结果，从而完成整个攻击过程。

### 2. 瞬态执行的攻击类型分类（Spectre-type Misprediction vs. Meltdown-type Fault）

瞬态执行攻击的一个共同点是它们都需要使用瞬态指令在微架构层面编码/访问未授权的数据，并且这些瞬态指令的执行结果永远也不会被提交到 ISA 层面，但是可以通过不同的渠道泄露这些信息。

攻击主要分为两类：第一类是利用数据流或控制流的**错误预测**（Misprediction）实现的攻击（在预测错误的分支下执行瞬态指令），该类型的攻击叫做 **Spectre**；另一类是利用出错指令（Faulting Instruction）实现攻击（在出错指令后部署瞬态指令），该类型的攻击叫做 **Meltdown**。这两个类型的攻击所利用的 CPU 特性不同，因此，它们的攻击范围和防御方法也各不相同。

- **Spectre** 类型的攻击是指：瞬态指令只能访问/计算应用可以访问的数据，因此，它可以在瞬态执行中绕过软件定义的安全策略（例如：边界检查、内存写入等），从而把程序中的秘密信息泄露出去。也就是说，攻击者可以诱导（Steering）受害者在瞬态执行过程中访问/计算只有受害者才能访问得到的数据（攻击者直接访问不到的数据），因此，这种类型的攻击还需要在受害者代码中找到可用的代码片（Code Gadget）。

- **Meltdown** 类型的攻击是指：跟随在出错（Fault）指令之后的瞬态指令可以突破 ISA 层面的隔离屏障、绕过硬件安全策略（Hardware-enforced Security Policies）去访问/计算应用程序在 ISA 层面无法访问到的数据（比如：内核数据等）。因此，该类型的攻击相比于 Spectre 类型的攻击更加强大。

#### (1). Spectre 类型攻击（Misprediction）

![Spectre 攻击类型的分类 1](/images/2019-11-13/Spectre_type_attack_classifiction1.png)

如表 1 所示是基于 Spectre 攻击所利用的**微架构元素**的不同进行的分类（一级分类，对应图 1 中的第一级分类：Spectre-type），像 NetSpectre、 SGXSpectre 和 SGXPectre 攻击都是利用表中的其中一种变体实现的。

![Spectre 攻击类型的分类 2](/images/2019-11-13/Spectre_type_attack_classifiction2.png)

如图 3 所示是基于 Spectre 攻击所毒害（Poison/Mistrain）**微架构分支预测器的策略**不同进行的分类（二级分类，对应图 1 中的第二级分类：Spectre-type），根据毒害预测器时的地址位置是否相同（在目标分支上进行毒害训练，或者在其它分支上进行毒害训练）、或者进程是否相同（在攻击者进程中进行毒害，在受害者进程中使用）进行分类，该分类不包含 Spectre-STL。**相同 CPU 核之间共享相同的预测器**。

##### (a). Spectre-PHT (Input Validation Bypass)

PHT（Pattern History Table）是用来预测条件分支是否进行跳转使用的，而 PHT 又是基于 BHB（Branch History Table）和一些额外的信息来决定的，最终的 BHB 是根据最近的 N 次跳转决定的（作者没给出 N 的值），因此，攻击者可以毒害 PHT 来实现攻击。

**瞬态越界读写**：

下面以越界读（Reading Out-of-Bounds）为例进行解释:

```
  if (x < len(array1)) 
  { 
    y = array2[array1[x] * 4096]; 
  }
  else
  {
    //...
  }
```

如上代码片所示，正常情况下，`x` 的值是不会超过 `array1` 数组的大小的，然而，如果一直给这个代码片提供一个合法的 `x` 值，PHT 就会预测这个跳转为 `true`，当后续执行这段代码的时候，CPU 就会根据之前的信息预测当前分支（我们称之为：**训练**），并进行预测执行。当攻击者再向这段代码片输入一个非法的 `x` 值的时候，CPU 就会按照之前训练的结果，错误的预测这个跳转为 `true`，并执行一个**瞬态的越界访问**。以上的这段代码就是一个信息泄露的微架构隐蔽信道，根据攻击者所提供的 `x` 值，瞬态执行过程中会把 `array2` 数组中的对应内存页载入到内存中，攻击者就可以根据 Timing 技术从隐蔽信道的接收端恢复出对应的秘密信息。

同样，也可以利用该技术进行越界写等攻击，实现瞬态控制流劫持（覆盖函数返回值等），代码片如下所示：

```
  if (x < len(array)) 
  { 
    array[x] = value; 
  }
  else
  {
    //...
  }
```

**被忽略的错误训练策略（作者新发现）**：

目前已经发现的 Spectre-PHT 攻击类型都是：相同地址空间（Same-Address-Space）的原地（In-Place）训练类型，但是根据作者的分类，还可以进行跨地址空间（Cross-Address-Space）的异地（Out-of-Place）训练类型等三种类型的变种攻击（图 1 中的右上角的四种攻击类型），并且分别在 ARM、Intel、AMD 处理器上都成功执行攻击，使得这种 Spectre-PHT 类型的变种攻击可以在攻击者进程中训练分支预测器，并在受害者进程中进行攻击。**例如**：对于 Intel 的 SGX 远程验证（Attestation）功能，攻击者可以在自己的 Enclave 里面训练预测器（与受害者具有相同的分支代码片：Congruent Branch），然后在受害者 Enclave 中使用这个预测器，达到 Enclave 中的瞬态执行攻击（**训练和使用预测器需要在同一个物理 CPU 核中**）。

##### (b). Spectre-BTB (Branch Target Injection)

BTB（Branch Target Buffer）是 CPU 用来预测分支跳转目标地址的，如果攻击者对 BTB 进行毒害（Poison），就可以使得 CPU 在瞬态执行期间跳转到任意的地址，如果结合 ROP（Return-Oriented Programming）技术，还可以实现瞬态控制流的劫持。

原始的 Spectre-BTB 攻击可以通过**跨地址空间原地**毒害 BTB 实现攻击，而 SGXPectre 攻击可以通过**跨地址空间原地**或**原地址空间异地** BTB 毒害技术从 Intel Enclave 中泄露秘密信息，作者根据分类树中的分类，发现了**原地址空间原地毒害** BTB 攻击技术（Spectre-BTB-SA-IP）。

##### (c). Spectre-RSB (Return Address Injection)

RSB（Return Stack Buffer）是 CPU 用来存储最近的 N 个函数返回地址的缓冲区，当 CPU 遇到一个 `ret` 指令的时候，它就会从 RSB 的栈顶弹出一个地址并根据这个地址进行瞬态的预测执行。因此，可以通过毒害 RSB 来实现瞬态的返回控制流劫持（例如：瞬态沙盒逃逸等）。

##### (d). Spectre-STL (Speculative Store Bypass)

CPU 的预测执行不仅仅是针对控制流的预测执行，也还会对数据流进行预测执行。STL（Store to Load）依赖关系指的是：执行 `Load` 指令之前，必须完成前面指令对该内存地址的写入操作。CPU 有一个内存预测器（Memory Disambiguator），当它知道前面所有 `Store` 指令的写入地址的时候，预测器就会预测那些 `Load` 操作可以预测执行。内存预测器会把与前面的 `Store` 指令不存在依赖关系的 `Load` 指令预测执行，把数据载入到 Cache 中，当预测器发现有冲突的时候（比如：前面的 `Store` 指令的地址和当前 `Store` 指令的地址出现重叠（Overlap）的时候），CPU 就会废掉预执行的结果，并重新执行 `Load` 指令，因此，很容易就产生“读到脏数据”这种事情。

#### (2). Meltdown 类型攻击（Fault）

Spectre 类型的攻击依赖的是对 CPU 内部的分支预测器进行误导性的训练，使其按照攻击者设定的方向进行预测执行而导致的，而 Meltdown 类型的攻击不同，它利用的是 CPU 产生异常的指令的后续指令（在地址空间上紧跟在产生异常之后的那些指令）来执行瞬态攻击。在某些微架构中，一条即将会产生错误的指令的后续指令可以被 CPU 预执行，因此，这些预执行的指令就有可能会对一些未授权访问的数据进行访问等操作，当 CPU 执行到这条特殊的指令（Trigger Instruction）的时候，就会产生一个错误，从而使得 CPU （CPU's in-order instruction retirement mechanism）把预执行的指令所得到的结果逐一丢弃，使得 ISA 层面感知不到，但是这些结果可以通过微架构隐蔽信道泄露出去。

Intel 对计算机中的异常分为三类：**Faults（故障）**、**Traps（陷阱）** 和 **Abort（终止）**，如果 CPU 在执行过程中遇到一个可以纠正的异常（比如：缺页等），而该异常被处理之后还可以是的程序继续往下执行（引发异常的指令会被重新执行），这种情况称为： Faults；Traps 与 Faults 不同的是，在引发异常之后，程序计数器会指向异常处理的开头，并且处理完异常之后，引发异常的指令不会被重写执行（比如：下断点等）；Abort 是指程序遇到了无法修复的错误，使得程序无法继续往下执行。**而之前的所有 Meltdown 类型的攻击都是由于 Faults 导致的**。

作者根据 Meltdown 类型进行分类，第一类是按照缺页异常保护位进行划分，分为 7 类，如表 3 所示：
![Meltdown 攻击类型的分类 1](/images/2019-11-13/Meltdown_type_attack_classifiction1.png)

第二种分类是按照存储位置，以及是否跨权限进行分类，分类结果如表 4 所示：
![Meltdown 攻击类型的分类 2](/images/2019-11-13/Meltdown_type_attack_classifiction2.png)

##### (a). Meltdown-US (Supervisor-only Bypass)

通常情况下，用户态的代码无法访问内核态的数据或代码，但是通过 Meltdown 攻击，可以间接的访问当操作系统的整个内核空间的数据，具体的例子如下所示（例子来源：[OSUSecLab-zhihu](https://zhuanlan.zhihu.com/p/32654221)）：

```
1  ; rcx = kernel address
2  ; rbx = probe array
3  mov al, byte [rcx]
4  shl rax, 0xc
5  mov rbx, qword [rbx + rax]
```

RCX 指向的是内核地址，代码中的第三行表示用户态的代码试图从内核空间中读取一个字节的数据到 `al` 中（这行代码也是 Meltdown 攻击的触发指令），因此会导致异常产生，但是由于 CPU 的乱序行机制，实际上，在微架构层面，第四和第五行代码都已经被预执行过了，而在预执行过程中，对数据的访问不受 CPU 权限的限制，因此，最后两条指令的数据其实来源于内核，第四行会把数据乘以 4096，也就是内存中一个内存页的大小，使得 CPU 在执行第五行的时候把探测数组（代码中的 RBX 所指向的数组）的其中一个内存页载入到 Cache 中（假设探测数组的每一项大小都是一个内存页大小），此时，攻击者就可以通过 Cache 的侧信道攻击来探测刚才载入的是哪一个内存页，从而推断出 RCX 寄存器所指向的内核地址中的一个字节的内容。

##### (b). Meltdown-P (Virtual Translation Bypass)

略（如有需求，请阅读原文）

##### (c). Meltdown-GP (System Register Bypass)

略（如有需求，请阅读原文）

##### (d). Meltdown-NM (FPU Register Bypass)

略（如有需求，请阅读原文）

##### (e). Meltdown-RW (Read-only Bypass)

略（如有需求，请阅读原文）

##### (f). Meltdown-PK (Protection Key Bypass)

Intel Spylake-SP 服务器 CPU 上有一种[用户态的内存保护机制](https://software.intel.com/en-us/articles/intel-xeon-processor-scalable-family-technical-overview)（Memory Protection Key for user space, PKU，其实应该就是`金同学`玩的那个 MPK）。PKU 允许进程在不调用任何系统调用的情况下，在用户态设定内存页的访问权限，使得用户态的应用程序可以实现高效的硬件支持（Hardware-enforced）的隔离功能。

当攻击者可以在受害者进程中执行代码的时候，攻击者就可以利用 Meltdown-PK 攻击目标进程，实现绕过 PKU 进行读写操作。而 Intel 对 PKU 的缓解方案是使用地址空间隔离。

##### (g). Meltdown-BR (Bounds Checks Bypass)

BR（Bound-range-exceeded Exception）是 X86（IA-32） 中引入的用于检查数组索引是否越界的一个机制（使用 `bound` 指令做检查），但是在后来的 X86-64 中又取消了这个机制，并使用内存保护扩展 MPX（Memory Protection eXetensions）来实现更有效的数组越界检查。

以前有人把这种基于 MPX 的边界检查绕过攻击分类为 Spectre 类型的攻击，但是在这篇文章中，作者证明了这种分类方法是错误的，作者证明了这种攻击时一种利用**延迟异常处理**的 Meltdown 攻击。并在 IA-32 架构上（包括 Intel 和 AMD 处理器）成功绕过 bound 指令，在 x86-64 架构上（仅 Intel 处理器）成功绕过最先进的 MPX 技术，并且是目前为止首个利用延迟异常处理技术在 AMD 处理器上实施 Meltdown 类型的瞬态执行攻击。

##### (h). Residual Meltdown (Negative Results)

略（如有需求，请阅读原文）



### 3. 代码片（Gadget）分类

作者把 “Gadget” 定义为一连串的指令集合，并根据前文中的图 2 来定义攻击过程中的各个阶段，把 Gadget 分类如表 6 所示，需要注意的是，在准备阶段（Preface），Gadget 的选择既可以来源于受害者的代码（例如：用于沙盒逃逸攻击的 Spectre-PHT），也可以由攻击者自己构造（例如：Meltdown-US），视情况而定。

![Gadget 分类](/images/2019-11-13/Gadget_classification.png)

目前的 Meltdown 类型的攻击普遍是用来 dump 目标进程或者内核的内存信息，而 Spectre 类型的攻击主要是在比较苛刻的条件下进行（Controlled Environment），并且，在现实世界中的 Spectre 类型攻击所面面临的最大的挑战就是如何去寻找可以利用的 Gadget。目前有利用 Taint Tracking 技术自动化的寻找 Gadget 的工具（例如：[oo7](https://arxiv.org/pdf/1807.05843.pdf)），也有利用符号执行查找 Gadget 的工具（例如：[Spectetor](https://arxiv.org/pdf/1812.08639.pdf)），还有 Linux 内核开发团队基于 [Smatch Static Analysis Tool](https://lwn.net/Articles/752409/) 开发的工具等。

学术界对 Spectre 类型的 Gadget 分析的文章仅有 5 篇，分析结果如表 8 所示：

![Spectre 类型的 Gadget](/images/2019-11-13/Gadget_of_spetre_type_attack_in_realworld.png)

### 4. 防御

#### (1) Spectre 类型的防御

作者把 Spectre 类型的防御分为 3 类：

- C1: 缓解或减少使用隐蔽信道窃取秘密信息的准确度（Accuracy）

  既然 Spectre 攻击是利用隐蔽信道（Covert Channel）来传送秘密信息，因此，就可以通过降低隐蔽信道的数据传送精度，但是这种防御方法并不是有效的防御方法。目前已有的一些在硬件层面的缓解方案有：SafeSpec（在指令执行预测失败的时候，使用硬件机制来抹除在瞬态执行过程中，在微架构层面产生的信息， SafeSpec 仅针对 Cache）、InvisiSpec 等，在软件层面，由于攻击者恢复信息的时候需要测量访存的时间来获取秘密信息（例如：Cache 命中或缺失等），因此，可以降低攻击者使用 Timer 的精度，比如在浏览器中的 JavaScript 中 Timer，可以防止基于浏览器的 Spectre 攻击。

- C2: 缓解或终止瞬态执行过程中对可访问数据的猜测

  由于 Spectre 类型的攻击都是依赖于 CPU 内部的各种预测机制，因此，有人就提出了一种有效放着这种类型攻击的方法：当程序访问敏感数据的时候，禁用 CPU 的各种预测机制。在硬件层面，Intel 和 AMD 都扩展了 ISA，实现了对 CPU 内部预测器的特殊控制（例如：可以在特定场合使得预测器免遭外部影响），ARMv8.5 的指令集中引入了一个新的屏障（`sb`）来限制预测执行等；在软件层面，Intel 和 AMD 都提出序列化执行指令（`lfence`），ARM 也提供了一种数据同步屏障（Data Synchronization Barrier，`DSB_SY`）和指令同步屏障（`ISB`）来防止预测执行。

- C3: 保证秘密信息不被访问

  比如像通过越界读来实现信息泄露的攻击方法，WebKit 提出了两种解决方案，一种是使用 MASK 来替代原来的越界检查机制，另一种是使用一个伪随机数来保护指针，还有谷歌也提出了一种 Site-isolation 技术防止跨进程空间读取数据，该技术在 Chrome 中默认被开启。

#### (2) Meltdown 类型的防御

作者把 Meltdown 类型的防御分为 2 类：

- D1: 确保 ISA 层面无法访问到的数据在微架构层面也无法访问

  Meltdown 类型的攻击所产生的根本原因是因为：CPU 允许瞬态执行指令访问 ISA 层面所无法访问的数据。因此，在出现 Faults 的时候防止访问未授权的数据是解决问题的关键，现在的 AMD 处理器和 Intel 处理器（Whiskey Lake）已经支持这项功能，另外，通过 Linux 内核页表隔离机制（KPTI）实现的 KAISER （防止内核页被映射到用户空间）本来是用来防御攻击者攻破 KASLR 的，但是它也可以用来防御部分的 Meltdown 类型的漏洞（例如：Meltdown-US），微软从 Windows 10 Build 17035 和 Mac OS X 也提供了类似的补丁。

- D2: 防止 Faults 的发生

  Meltdown 类型的攻击所利用的是 CPU 的延迟异常处理功能，因此，如果不让 Faults 产生，也就不会出现这种类型的攻击，典型的例子是：当外部访问 SGX 内部页的时候，Enclave 直接返回 -1， 而不是产生错误（Faults），因此，SGX 天生就可以防御 Meltdown-US 类型的攻击。

## 四、实验评估

### 1. 实验环境

- CPU : 
  - For Intel : Skylake i5-6200U, Coffee Lake i78700K and Whiskey Lake i7 8565U
  - For AMD : Ryzen 1950X and Ryzen Threadripper 1920X
  - For ARM : NVIDIA Jetson TX1
  - For Cloud : Amazon EC2 C5 with Ubuntu 18.04 & PKU support
- Memory : -
- OS :  -
- Kernel Version : 4.19
- Tools : -

### 2. 实验结果

#### (1). 实现世界中的 Gadget 分析

作者选择了 5.0 版本的 Linux 内核来评估（作者很隐晦的避开了评估的方法）所存在的 Spectre-PHT 攻击所需要用到的 Gadget 数量，评估结果如表 7 所示，表中的 Gadget 分为四类:

- Prefetch: 对单个数组的访问（需要结合其它 Gadget 才能发挥作用）。
- Compare: 加载数据并进行跳转（不是很明白）。
- Index: 对双数组的访问，是一种比较有效的 Gadget，但是实验中并没有发现这样的 Gadget。
- Execute: 加载数据到数组中，然后利用加载进来的数据当做函数执行进行跳转。

一般而言，表中的索引 `i`  的值是需要能够被攻击者所控制才行。

![Linux 内核中 Spectre-PHT 类型的 Gadget 数量](/images/2019-11-13/Spectre_PHT_gadget.png)

#### (2). 防御评估

作者在三大主流处理器上分别评估了前文所提及的各类攻击（现存的），并在理论层面评估了需要修改硬件实现防御的方法。

首先是 Spectre 类型的防御方案评估，如表 10 所示，表中的左边表示的是各个 CPU 类型，以及在改 CPU 类型上的攻击方法，上面是对应防御方案，表中的实心圆表示该攻击类型已经被缓解，半实心圆表示部分缓解，空心的圆表示未缓解，而方形表示从理论层面的缓解情况。表中的斜体表示的防御方案表示可以该缓解方案可以投入实际中使用（**Production-Ready**），其它的表示该防御方案仅存在于学术层面上。棱形表示不再讨论范围。图中的 [Taint Tracking, S&P'19](https://spectreattack.com/spectre.pdf) 防御方式是指 CPU 跟踪瞬态执行过程中的数据载入，并可以防止它进入隐蔽信道（怎么防止作者没说，只是给了一个引用），因此，作者说这个防御方案可以在理论层面防御所有类型的 Spectre 攻击。

![Spectre 类型的防御方案](/images/2019-11-13/Spectre_defenses.png)

对于 Meltdown 类型的攻击，作者的评估方法是在一个已经完全打补丁的系统上执行 Meltdown 攻击，并观察该系统是否真的能够防御相应的 Meltdown 攻击，测试的环境包括上文【四.1】中提及的所有处理器（包括一个亚马逊的云），测试结果显示，作者任然可以在部分处理器上执行攻击，如下表所示：

| 平台                                                         | 攻击类型                                  |
| ------------------------------------------------------------ | ----------------------------------------- |
| Ryzen Threadripper 1920X                                     | Meltdown-BND                              |
| Skylake i5-6200U, Coffee Lake i78700K and Whiskey Lake i7 8565U | Meltdown-MPX、 Meltdown-BND、 Meltdown-RW |
| Amazon EC2 C5                                                | Meltdown-PK                               |

#### (3). 对策的性能影响

作者对部分应对措施进行了性能评估，由于有些应对策略需要修改硬件，因此，作者没有评估那些类型的应对策略，作者所选择要评估的应对策略如表 11 所示，表中的上半部分是对现实软件所进行的评估，而下半部分是对特定的 Benchmark 进行的评估。

![防御措施所对应的性能影响](/images/2019-11-13/Performance_impact.png)

表中有两个地方的数据比较引人注目，第一个是**序列化执行**（Serialization）防御措施，这种防御措施理所当然的会大幅度的降低系统的性能，也就是使得 CPU 没法进行预测执行了；另一个笔记有意思的是 **KAISER/KPTI**，据官方的报告是说，这项防御措施会对性能产生较大的影响，但作者的评估结果中却显示该方案对性能几乎没有影响。

此外，[Larabel](https://www.phoronix.com/scan.php?page=article&item=linux-419-mitigations&num=1) 对 Linux 4.19 进行了一个更加系统的评估，他把系统中所有的 Meltdown 和 Spectre 防御方案都打开和关闭进行两组对立的测试，测试结果表明，在 **Intel** 处理器上的性能下降约为：**7-16%**，而在 **AMD** 处理器上的性能下降约为：**3-4%**。

最后，对于性能和安全性之间的权衡，作者说道：对于大部分用户而言，以上的这些漏洞被利用的风险是很低的，并且 Linux、Misoft 和 Apple 的默认软件防御措施已经足够了，并且这些默认的方案已经是性能和安全性之间的最优选择了，但是对于数据中心来说，就要视情况而定，不同安全等级的数据中心所选择的方案不尽相同。

## 五、优缺点

### 优点：

- 正确的给 Meltdown 和 Spectre 类型的攻击进行分类，并纠正了以前分类中出现的一些不正确的、误分类的情况。
- 文章比较全面的总结了 Meltdown 和 Spectre 类型的攻击，并给予了对应的防御方式，包括硬件防御方式和软件防御方式，最后还系统的评估了各种防御方式所带来的性能开销。

### 缺点：

- 论文中没有指出各种 CPU 中所支持的最大瞬态执行指令数量，也没有给出作者在执行攻击的时候所执行的瞬态指令数量。
- 由于论文所涉及的知识点非常多，因此也就无法仔细分析每一个知识点的细节，大部分都只是带过，也没有对应的例子说明，因此，对于一般读者来说，比较难读懂。


## 六、总结和看法

整篇文章都围绕着瞬态执行攻击（Transient Execution Attack）进行展开，并解释了什么是瞬态执行（在 ISA 层面不可见，但是在微架构层面可见的指令执行方式），文中作者系统的对瞬态执行攻击进行分类和分析，并揭露了 **6** 个以前没有被人们所发现的瞬态执行攻击（2 个 Meltdown 类型和 4 个 Spectre 类型的变体）。此外，作者还对所有的攻击类型在三个主流的 CPU（AMD、Intel 和 ARM）上进行了概念验证（Proof-of-Concept），并在 Github 上给出了这些攻击的利用代码（POC），与此同时，还对产生这些漏洞的代码片（Gadget）进行了研究和分类，最后总结了各种攻击类型的防御措施，并评估了这些防御措施的性能开销等。

在我看来，这篇文章的内容太过丰富，量比较大，一下子难以吸收文中的所有知识点。如果需要往这方面继续研究，还需要阅读作者在文中引用的其它论文，这样才会更加深刻的理解每一个攻击方法以及对应的防御措施。
