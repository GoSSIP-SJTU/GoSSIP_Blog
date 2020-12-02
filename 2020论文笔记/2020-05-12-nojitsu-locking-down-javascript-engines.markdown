---
layout: post
title: "NOJITSU: Locking Down JavaScript Engines"
date: 2020-05-12 15:02:51 +0800
comments: true
categories: 
---

> 作者：Taemin Park, Karel Dhondt, David Gens, Yeoul Na, Stijn Volckaert, Michael Franz
>
> 单位：University of California, Irvine
> 出处：NDSS 2020
>
> 资料：[Paper](https://www.ndss-symposium.org/wp-content/uploads/2020/02/24262-paper.pdf)

## 1.  简介

浏览器解释引擎对代码重用攻击的防御措施变得越来越强，数据攻击也自然而然变得很流行。其中JIT编译器是一个很吸引攻击者的攻击目标。通过攻击JIT编译器的敏感数据，例如IR，可以达到代码执行的目的。因此现在针对JIT编译器的敏感数据保护是一个研究热点。

本文中作者发现目前的针对JIT编译器的敏感数据保护的措施还不够有效。于是作者设计了一个新的防御措施NOJITSU来保护解释器引擎里的敏感数据。

本文中作者的贡献：

1. 提出了一种新的数据攻击方法（针对Mozilla的SpiderMonkey解释器），从而说明目前的数据攻击防护措施还不够有效。
2. 设计了一个新的防御方案NOJITSU，通过Intel MPK对内存进行细粒度的访问控制。
3. 进行了实验评估，NOJITSU可以成功防御代码重用和数据攻击，并且运行时开销只有5%。

<!-- more -->

## 2. 背景

**JIT数据攻击利用的发展历史：**

1. 代码注入（2010年）：最开始没有任何保护，直接通过修改JIT代码页的代码，执行shellcode，之后JIT页有了NX保护，可以有效防御这种攻击。
2. JIT喷射（2010年）：通过脚本语言里的立即数，注入代码片段，可以绕过NX和ASLR，后来的Constant Blinding、Constant Elimination、Code Obfuscation、Code Randomization、CFI等防御措施可以有效防御。
3. 基于JIT的代码重用（2015年）：利用JIT生成的小gadget。需要跟JIT-ROP进行区分，JIT-ROP是与JIT编译器无关的，仅仅指locate, read，disassembly static。
code这个过程。因为不像ROP一样需要制定gadget地址，所以可以绕过NX和ASLR。需要用XOM（Execution only memory）来防御。
4. 对JIT编译器的攻击（2016年）：JIT编译器在工作时通常是先让某块内存页可写不可执行，然后往里面写入JIT生成的native code，然后再将其权限修改成可执行不可写。如果利用这个时间窗口，在这个代码区域还是可写不可执行的时候，写入自己想要的shellcode（通过其它的线程），就能绕过了。针对这类攻击，最近的防御措施是将编译过程和JIT代码的执行过程进行进程级的隔离，或者用基于硬件的TEE。

## 3. 攻击方法

**3.1 威胁模型**

攻击对象为Firefox浏览器使用的SpiderMonkey。

存在代码注入和代码重用的保护措施。硬件漏洞不在考虑范围内。存在内存破坏漏洞。

**3.2 SpiderMonkey**

SpiderMonkey的架构如下图所示。

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled.png](/images/2020-05-12/Untitled.png)

关键数据：

1. Bytecode Region：字节码opcode和operands。
2. JIT Code Cache：普通的JIT Code以及inline cache stubs。
3. JIT Compiler Data：JIT编译器的IR（SpiderMonkey里为MIR和LIR）以及一些其他的被JIT编译器使用的数据。
4. JS Data Objects：由垃圾回收器管理的管理的所有Object。
5. Object Tables

现有的保护技术只保护了JIT Code Cache和JIT Compiler Data，其他的数据都还是有攻击面的。

**3.3 任意地址读写**

通过CVE-2019-11707（类型混淆漏洞）获得任意地址读写，实际的利用分为四步：

1. 分配大量很小的ArrayBuffer，每个都是32字节。
2. 为每个ArrayBuffer都创建Uint32Array和Uint8Array这样两个View Object。
3. 触发类型混淆（Uint32Array和Uint8Array之间）。
4. 之后就可以对ArrayBuffer里的32个Uint32元素进行读写了。因为ArrayBuffer只有32个字节，所以可以修改到后面相邻ArrayBuffer的metadata（写入一个目标指针），这样就可以任意地址读写了。

**3.4 本文提出的利用方法**

获得任意地址读写能力后，如下图所示是完整的数据攻击利用流程：

1. 泄漏出JSContext和JSFunction地址
2. 修改JSFunction里的函数指针，指向要调用的目标函数（system()函数），将参数写在JSContext里。
3. 从JavaScript脚本里调用目标函数。

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%201.png](/images/2020-05-12/Untitled%201.png)

泄漏JSContext的地址以及目标JSFunction对象的地址的方法，如下图所示：

- 从被破坏的ArrayBuffer里读取NativeObject，然后顺着里面的指针信息找到全局的JSContext对象地址。
- 在NativeObject里写入一个目标JSFunction对象的引用，然后获得目标JSFunction对象的地址。
- 要找目标调用函数的地址，可以通过已经泄漏出来的JSFunction里的Native函数地址来找（通过PLT）。

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%202.png](/images/2020-05-12/Untitled%202.png)

这个方法目前不能被现有的针对JIT Compiler Data的保护所防御。因为攻击的目标是解释器。

## 4. NOJITSU设计

NOJITSU是一个细粒度的内存保护方案。由于解释执行和JIT代码执行的切换很频繁，所以不能简单的通过进程间隔离的方式进行隔离。NOJITSU通过自动化的动态分析和静态分析来限制内存访问权限，从而可以防御大量的攻击。NOJITSU和现有的一些防御方案可以兼容，比如Constant Blinding、Constant Elimination、 Code Obfuscation，Code Randomization，或者CFI。

如下图所示。

- JIT Code：JIT Code Cache包含里动态生成的指令，可以在CPU本地运行。为了防止代码注入，需要将JIT Code Cache的权限设置为不可写（除了生成这些指令的时候）。JIT Code Cache还需要被设置成不可读，以防止JIT-ROP攻击。但是JIT Code Cache本身是会被读的（例如不能被嵌入到指令里作为立即数的常量，以及跳转表）。为了让JIT Code只可执行，作者手动将这些数据进行了分离，然后只对代码设置了只可执行的权限。
- Static Code：解释引擎本身以及使用的依赖库的代码，NOJITSU将这些代码的权限设置为了只可执行，可以防JIT-ROP攻击。
- JIT IR：JIT IR是字节码到JIT代码的中间表示，IR的生命周期很短，现在已有针对JIT IR的攻击，从另一个线程利用条件竞争来破坏IR。NOJITSU对JIT IR进行了保护，只在当前线程有写权限。
- Bytecode和Object Tables：这两个也只能在生成的时候可写，剩余的生命周期只允许读，所以NOJITSU只在解析脚本生成它们的时候给予写权限。
- Javascript Objects：Javascript Objects不同，不是只在生成的时候可以被写，这些对象会在整个生命周期里被频繁进行写，比如一个包含程序变量的Data Object，在程序执行的任何时候都有可能被写，甚至这些对象里的一些flag也会被频繁更新（譬如引用计数器）。因此要找出程序执行中的所有对这些对象进行写的位置是很困难的。NOJITSU采用的是动态分析方法。

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%203.png](/images/2020-05-12/Untitled%203.png)

作者将JavaScript Objects分成里两种保护域：

- Sensitive Data Objects：包含函数指针、Object Shape Metadata、Scope Metadata、JIT Code这样的敏感信息的Object。对这类对象的破坏可以让攻击者直接获得控制权。
- Primitive Data Objects：包含整数、字符、数组的对象，对这些对象的破坏不能直接劫持控制流，但是可以帮助接下来破坏Sensitive Data Objects。

对这两类敏感数据的分离，可以保证这些对象不能在同一时间可写。对于之前的例子来说，攻击者也将不能通过利用类型混淆来通过从Primitive Data Objects里获得的任意地址读写来破坏Sensitive Data Objects。

需要注意的是，这些对象在JIT Code执行的时候也会被频繁写，在JIT Code执行时修改Object的权限会带来运行时的开销（因为JIT Code是高度优化的）。因此在JIT Code执行期间，NOJITSU会完全打开对Primitive Data Object的访问。

## 5. NOJITSU实现

将NOJITSU用于保护SpiderMonkey 60.0，作者修改了内存分配代码部分的源码，将对应的domain key分配给不同类型的结构体对象，然后对所有的需要对字节码、对象表、JIT IR、JIT代码、JIT数据进行访问的代码进行了插桩，来保证有足够的权限进行访问。为了隔离JIT代码和JIT数据，作者修改了JIT链接器以及汇编器代码。最后修改了SpiderMonkey的信号处理句柄，以支持自动化地插桩。

动态分析的方法如下图所示：

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%204.png](/images/2020-05-12/Untitled%204.png)

**优化：**PKRU寄存器的写需要20个时钟周期，虽然对比mprotect来说已经好了很多，但是过于频繁的写还是会带来很高的开销。所以作者是对整个Accessor Function（需要写权限的函数）进行了保护。如果PKRU寄存器的更新是在一个被频繁调用的代码区域内，譬如一个循环，或者构造函数析构函数里，性能影响会很大，所以作者进行了优化，把整个PKRU寄存器的更新移到里这类代码区域的外面。例如如下的代码，通过作者这种方式的优化，可以显著提升性能。

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%205.png](/images/2020-05-12/Untitled%205.png)

这里作者用启发式的方法将那些会被频繁调用的函数选出来。先提取SpiderMonkey的全局Call Graph，然后作者对CallGraph里的每一个函数进行评分。评分方法如下所示，叶节点的Accessor Function，如果需要访问，分数设置为1，不需要访问则设置为0，然后计算其他非Accessor Function的评分。评分高于0.15的函数都会被赋予权限。权限切换的指令插入作者也进行了优化，没有直接放在函数调用前后，而是通过支配分析，选择能到达所有需要写权限的基本块的基本块，插入set_pkru。

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%206.png](/images/2020-05-12/Untitled%206.png)

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%207.png](/images/2020-05-12/Untitled%207.png)

## 6. 实验评估

**6.1 安全性**

目标是防止代码注入、代码重用、数据攻击。核心思想是限制整个程序生命周期内对目标变量的访问权限。

保证修改后的程序能正确允许：用SpiderMonkey的内建测试用例（包含里6000个测试脚本）来进行插桩的确定。

如下图所示是作者进行的数据统计，两类Object在解释引擎里需要写权限的函数所占总函数的比例。

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%208.png](/images/2020-05-12/Untitled%208.png)

如下图所示是在整个写的窗口里，实际的写指令所占的比例。

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%209.png](/images/2020-05-12/Untitled%209.png)

Code-Injection Attacks：主要为防御JIT Spraying Attack。现有的Constant Blinding开销比较高，SpiderMonkey并没有采用。NOJITSU可以解决这个问题，因为NOJITSU已经把JIT Code里的常量数据分离出来，并且设置为了不可执行不可写，所以也没办法在常量数据注入代码了。

Code-Reuse Attacks：为了验证NOJITSU能防止JIT-ROP攻击，作者针对CVE-2019-11707这个漏洞写里完整的JIT-ROP的利用程序，这个利用程序对于没保护的SpiderMonkey是可以可靠的进行利用的，对于保护后的SpiderMonkey则没法进行利用，因为没办法运行时反汇编gadget。

Bytecode Interpreter Attacks：就是本文作者提出的攻击方法。现有的防御方法都没有办法阻止这种攻击，但是NOJITSU可以（因为这也是NOJITSU提出来的目标之一），NOJITSU将Sensitve Data Objects和Primitve Data Objects进行了分离（Function Objects属于Sensitive Data Objects）。 当然除了解释器里的对象，Bytecode、Data Objects、Data Tables、JIT Compiler Data也用不同的MPK域进行了隔离。

动态分析的覆盖率：作者将测试用例分为了两部分，相当于一部分拿来训练，一部分拿来测试训练的结果。训练用的脚本有6246个，测试用的脚本有30605个。这6246个脚本训练出来的SpiderMonkey能顺利运行30605个测试用的脚本。这说明动态分析已经足够健壮了。

性能：测试环境Intel Xeon Silver 4112，2.60GHz CPU，32GB内存。在Ubuntu 18.04.1下跑benchmark。Benchmark用的是LongSpider。测试结果如下图所示，整体开销不超过5%，开优化选项后为2%。

![NOJITSU%20Locking%20Down%20JavaScript%20Engines%203c970e35b08c44ceb549db76dd579686/Untitled%2010.png](/images/2020-05-12/Untitled%2010.png)

## 7. 总结

这个工作提出了一个新的对SpiderMonkey的数据攻击思路，可以绕过现有的所有防御方案，通过人工对SpiderMonkey的数据进行隔离，并配合动态分析的反馈，将SpiderMonkey进行了完整的保护。
