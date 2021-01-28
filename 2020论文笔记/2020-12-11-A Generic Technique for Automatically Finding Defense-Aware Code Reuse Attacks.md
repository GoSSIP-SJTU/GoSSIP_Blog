#  A Generic Technique for Automatically Finding Defense-Aware Code Reuse Attacks

> 作者：Edward J. Schwartz<sup>1</sup>，Cory F. Cohen<sup>1</sup>, Jeffrey S. Gennari<sup>1</sup>, Stephanie M. Schwartz<sup>2</sup>
>
> 单位：Carnegie Mellon University，Millersville University
>
> 会议：CCS 2020
>
> 论文链接：https://edmcman.github.io/papers/ccs20.pdf



## Abstract

过去十年里，出现了大量针对代码重用攻击的研究。这项研究起源于早期对 ROP 攻击的研究，该攻击表明当时新提出的不可执行内存（NX）保护是可以被绕过的。在那之后防御者提出了针对 ret 指令的防御措施来防止 ROP攻击，而后攻击者又实现了 JOP 等实现等价ret 指令的攻击方法。最近几年又一些团体提出了新的防御措施防止代码重用攻击，比如控制流完整性 CFI 的检测，之后又有了对这些防御方式的攻击，比如对于 CFI 保护的绕过方法 Data-Oriented Programming。defense-aware attack 是指根据防御策略来实现特定的攻击，作者提出了一个通用的框架来实现 defense-aware attack。核心观点是，代码重用攻击可以被定义为状态可达性问题，而各种防御策略是防止状态之间的某些转换。作者实现一个名为Limbo的工具，并且对Limbo进行了评估。评估结果表明当可供重用的代码很少时，Limbo表现出色，使其成为现有技术的补充。另外，尽管Limbo没有关于ROP或DOP攻击的针对性知识和设计，但它的性能优于自动化ROP攻击的现有系统（angrop），以及针对细粒度CFI的保护下自动化DOP攻击的系统( BOPC )。




## Background

### Code Reuse Attacks

* **Non-Executable Memory 及 ROP**

  ​		早期的控制流程攻击包括两个步骤。攻击者首先将 shellcode 注入目标进程的内存空间。然后，攻击者会使用一个漏洞将控制流传输到她注入的代码。此时，攻击者可以在脆弱程序的上下文中执行任意指令。所以为了防御这种攻击，有了 NX 保护机制去避免可写可执行的内存空间。

  ​		随着NX的引入，攻击者认识到，虽然他们不能将新代码引入程序，但他们仍然可以将已经存在的代码用于其他目的，这种攻击称为代码重用攻击。最常用的代码重用攻击是 ROP 攻击，攻击者在内存中搜索以ret结尾的小的指令序列（gadget），并将这些gadget串在一起以实现他想要的命令执行。

  **Example**

  * 通过栈溢出漏洞将函数调用返回时的栈布置为 

  ```
  rsp     -> 0x40080 (pop rdi;ret)
  rsp+8	-> 0x60088 (the address of "/bin/sh")
  rsp+16  -> 0x40180 (the address of function system)
  ```

  ​		实现劫持控制流并执行 system("/bin/sh")

  

* **Code Reuse Defenses and CFI**

  ​		自针对 NX 防御措施的 ROP 攻击被开发以来，已经出现了好几次代码重用攻击与防御的相互促进。一些最早的防御专门集中在ret指令上（针对ROP攻击），直到攻击者提出了一种不需要使用ret指令的新技术，比如面向跳跃编程(JOP），其背后的想法是找到那些在 ret 之外的指令序列中，但在语义上等价的 gadget ，比如pop eax;jmp eax。在这种情况下，ret指令不再需要，这使得针对 ROP 的防御被绕过。

  ​		但在这之后防御者又提出了控制流完整性 (CFI) 检测，其核心思想是检测程序的执行路径何时偏离程序的静态控制流图(CFG)。当程序执行到程序 CFG 不允许的执行路径时，将终止程序。CFI 检测具体实现被非常广泛地分为粗粒度CFI或细粒度CFI。
  
  ​		为了应对 CFI 防御，攻击者又开发了多种攻击策略，既针对粗粒度的，也针对细粒度的CFI。这些技术包括Call-Oriented Programming (COP)、Data-Oriented Programming (DOP) 和 Counterfeit Object-Oriented Programming(COOP)。



### Concolic Execution

​		协同执行是具体执行和符号执行的结合。其从具体执行开始，沿着与具体执行相同的程序路径符号化地执行程序，并根据符号化的路径生成更多的测试路径。

* Symbolic Execution

  ​		假设当前执行路径描述为 a，如果遇到一个分支需要满足 EAX > 42, 那么生成符号 S<sub>eax</sub>  > 42，并将该条件加入执行路径,即更新描述符为 a ^  (S<sub>eax</sub>  > 42)

* Get new case

  ​		尝试将符号路径中的分支进行翻转来生成新的用例。即 SMT 求解 a ^  ! (S<sub>eax</sub>  > 42) 路径可能存在，那么将生成新的测试用例。 



## Limbo

​		代码重用攻击可以定义为程序从一个漏洞开始执行到攻击者所期望的目标状态。这个定义中的漏洞和目标状态通常是已知的，因此主要的问题是确定是否存在这样的程序执行路径，如果存在，如何触发。

### Vulnerable Starting State

​		Limbo允许用户通过两种方式指定漏洞或启动状态。

* 用户提供一个触发控制流漏洞的具体测试用例，Limbo会自动检测控制流漏洞的起点，实现对程序控制流的部分/任意控制。

* 用户提供一个不会触发漏洞的具体测试用例。在这种情况下，Limbo 会像常规的协同执行器不断生成新的测试用例，从而寻找控制流漏洞。一旦发现了一个控制流漏洞，就和上一条情况等效。



### Goal States

​        Limbo允许用户通过提供一个目标表达式来指定目标状态，该目标表达式可以是指定寄存器和内存值。当且仅当目标表达式在该状态中求值为真时，该状态就是目标状态。

* M[0x60080] = 0xdeadbeef
* %edi = 1    &   %eip = 0x40080                  (*0x40080)(1)

目标条件是定义攻击者目标的一种简单和有用的方法。因为协同执行维护路径, 它很容易检查是否执行达到一个目标。Limbo 查询 SMT 约束求解器是否可以满足 a∧expr。如果是，Limbo已经找到了一个达到目标状态的执行，并且将创建相应的测试用例。



### Exploring Reachable States

​		Limbo 将从用户输入到程序的漏洞处的符号路径作为起始状态集合，不断地进行协同执行生成新的状态并对是否达到目标状态进行判断，直到找到所求状态，或状态集合为空。但这显然会产生过量状态，因此作者采用了启发式的办法，其有两个准则：

* 选择能控制更多符号的程序状态。
  * 能控制 [esp+4] 与 [esp+8] 的情况下 
    * pop eax;ret;
    * ret;
* 选择尽可能小的状态空间 —— 尽可能降低符号间接跳转的数目。
  * jmp eax;

作者还采取了一些控制参数来减少过多的程序状态：

* max-branches：从上次符号跳转后的最大路径分支数（默认为0）。
* max-indjumps:  符号间接跳转的数目（默认为3）
* 间接跳转的代码区域 ：对于任意地址跳转，该工具允许用户设置目标代码区域，降低过多可能的跳转目标。



## EVALUATION

* **Can Limbo discover code reuse attacks in the presence of fine-grained CFI?** 

  作者用Limbo和已有的BOPC（针对 CFI 生成 DOP 攻击的自动工具）进行了比较：

  ![image-20201210230411027](/images/2020-12-11/image-20201210230411027.png)
  
  ![image-20201109220857627](/images/2020-12-11/image-20201109220857627.png)



* **How much executable code does Limbo require to produce code reuse attacks?**

  为了测试有效性，作者在一些已知的小项目代码中伪造了栈溢出漏洞并与 angrop 进行对比。

![image-20201109220908768](/images/2020-12-11/image-20201109220908768.png)

* **Can Limbo discover new techniques?**

  * 目标程序：tput

  * **start state** : 伪造的栈溢出漏洞

  * **goal** : M[0x804b000] = 42

    

  __libc_csu_init -> use_env@plt
  
  ![image-20201211010239909](/images/2020-12-11/image-20201211010239909.png)
  
  use_env
  
  ![image-20201211010251956](/images/2020-12-11/image-20201211010251956.png)
  
  
  
  %ebx = 0x804a5fc 
  
  0x804a5fc + 0xa4 = 0x804b000
  
  ![image-20201211010300808](/images/2020-12-11/image-20201211010300808.png)
  
  

## LIMITATIONS

* **Limitations of Concolic Execution**
  * 由于符号间接跳转会导致过多的路径，Limbo在小程序和受防御措施(如CFI)保护的程序上执行得最好，因为CFI限制了间接跳转时可能的状态转换。
* **Implementation Limitations**
  * 目前只支持32位x86 Linux可执行文件，但该设计方案是可以开发更多的架构支持。