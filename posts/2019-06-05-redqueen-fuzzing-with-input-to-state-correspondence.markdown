---
layout: post
title: "REDQUEEN: Fuzzing with Input-to-State Correspondence"
date: 2019-06-05 12:12:44 +0800
comments: true
categories: 
---

作者：Cornelius Aschermann, Sergej Schumilo, Tim Blazytko, Robert Gawlik and Thorsten Holz

单位：Ruhr-Universit¨at Bochum

会议：The Network and Distributed System Security Symposium 2019  

链接：https://www.ei.ruhr-uni-bochum.de/media/emma/veroeffentlichungen/2018/12/17/NDSS19-Redqueen.pdf
<hr/>

### Introduction
#### Smarter fuzzing
在过去几年，smarter的fuzzing无论在学术还是工业上都取得了很大进展。代表：afl。    
理想的smarter fuzzing基于一个最小的先验知识集合开始fuzz，但这和高效率的fuzzing所需要的两个假设相冲突：1.从一个好的seed input合集开始，2.拥有一个输入格式的生成器产生符合格式的input。afl使用反馈驱动的fuzzing方式来解决这个问题，它使用一个队列保留在fuzzing中产生新的code coverage的输入，从而“学习”到一个更有趣的输入集合。 

<!--more-->
#### Challenge   
但smarter fuzzer在发现新的代码路径的时候存在两个难以解决的问题，程序中magic number和checksum。例如，
在下面的例子中，只有前八个byte的输入符合“MAGICHDR"，并且输入的checksum与输入提供的吻合时，才能找到bug。    

![avatar](/images/2019-06-05/1.png)    

但对于随机生成和变异的算法来说，恰好产生符合条件的输入概率是可以忽略的。 

#### Old methods   
之前解决这个问题通常使用符号执行和污点追踪的方法。符号执行（修改符号）和污点追踪（污点传播）都需要对执行的平台和库函数的行为给出一些定义，具有一些局限性。

#### New one
作者给出了一种新的基于tracing 的方法，这种方法基于一个简单的假设。输入的一部分在运行时会直接传入到内存或寄存器中进行比较，在程序输入和当前状态上存在一种极强的输入-状态关联性。这个方法首先在程序执行时追踪并识别比较指令，然后判断特定位置输入的修改是否会对应程序状态（寄存器，内存）的修改，最后通过修改fuzzing中的输入确定修改是否给fuzzing带来了新的code coverage。    
作者基于这个方法实现了一个原型:REDQUEEN。其速度高于KLEE,VUzzer,AFLFast,并且找到了LAVA-M 数据集上比列出的bug更多的bug。同时REDQUEEN还发现了2个linux文件系统的bug和55个不同用户程序中的bug。

### Input to state correspondence
作者观察到，对于大量的程序来说,从输入中获取的值在执行过程中的大量状态中被直接使用。通过观察这些数据，可以猜测在哪些位置替换掉哪些值会bypass magic bytes和checksum的检查。

#### Magic bytes
一个典型的例子如下图。    

![avatar](/images/2019-06-05/2.png)     

这样的构造很难被反馈驱动的fuzzing工具所解决，因为fuzzer很难猜到一个符合条件的输入。
对于REDQUEEN来说，这个问题很容易解决。在每次发现一条新路径时，REDQUEEN hook所有的比较指令，当发现一个比较有“清晰的”参数，（例如 cmp a,b）就创建变异规则patten->repl(a->b,b->a)。
#####i）Tracing
在fuzzing一个新输入时，进行一次tracing，hook所有的比较指令（包括一些switch结构和比较函数）并提取参数。    
对上面的例子，输入为"TestSeed",在小端模式下得到cmp “deeStesT”,“RDHCIGAM”。

#####ii) Variation
对提取的参数进行一定的变化。
由于不知道比较条件的符合要求（lower or higher），对参数可以加1，减1。得到“RDHCIGAM”,“RDHCIGAL”,“RDHCIGAN”

#####iii) Encodings
对编码进行一定的变换。
同样进行大小端和其他一些模式的编码变换（zextend,reverse,c-string），得到“MAGICHDR”, “LAGICHDR”,“NAGICHDR”。

#####iv) Application
创建`patten->repl`的规则对输入进行应用。这样的mutation只发生一次，并且mutation规则使用的概率是100%。

#####v) Colorization and strings1
值得一提的是，作者使用了“着色化”的方法来减少能够匹配到的位置，简单的说就是在初始输入中增加熵，让初始输入更有随机性，更像"asdmohaoianxb",而不是"zzzzzzzzzzz"。
同时，作者对memcmp等函数也做了特定的特化处理。

#### Checksum
另一个fuzzer面对的挑战就是如何处理checksum。一个典型的例子如下图。    
![avatar](/images/2019-06-05/3.png)       
作者给出的方法是，首先1.识别所有看上去像checksum的比较。2.接着把这些比较patch掉，得到一个patched binary,在此之上进行fuzzing。3.在后面的fuzzing过程中，如果发现了有趣的输入可以到达新的路径，就把这些patch去掉，用处理magic bytes的方法替换，如果替换后能够产生同样有趣的输入到达同样的路径，那么继续fuzzing，否则就删去这个patch（因为替换无法实现，减少后面的替换工作）。
由于作者在unpatched binary上进行了验证，就没有了taintscope等工具有的false postive。

##### i）Identification
作者用下面的规则来筛出有关于checksum的比较。
i)在bypass magic bytes的过程中找到所有mutation规则左手边的pattern
ii)比较的两个参数都不是立即数
iii)在“着色”时，随着输入的变化，patten对应的repl也在变化。
这样的规则并不完善，可能带来一些false positive，但REDQUEEN在将一个新输入加入队列时，会删除所有相关的patch，可以避免false positive。

##### ii)Patch
在识别出所有可能的checksum后进行patch。

##### iii)Verification
在verification阶段，在fuzzing过程中生成的所有新输入在一个real target上用之前在magic bytes部分提到的规则进行修正，如果修正后的输入能够在real target上触发新的code coverage，就保留，否则就把对应的patch删除。

### Implementation details
#### kafl
作者基于kafl来实现REDQUEEN。作者在kafl上添加了大约10k行代码，大部分代码是为了添加ring 3 fuzzing的支持，添加对QEUMU-PT和KVM-PT的支持和修正一些bug。

#### Comparison hook
REDQUEEN使用硬件虚拟机断点来提取input to state correspondence,QEMU-PT本身提供了在执行时的反汇编功能，REDQUEEN在此时记录这些比较指令并在下次下断点并提取参数。断下的指令不仅仅是cmp，也包括call指令。

#### Colorization
REDQUEEN用下面的算法来进行“着色”，简单的说，就是尽可能尝试将更多的bytes在afl的bitmap（表示code coverage）不变的情况下替换成随机bytes。
![avatar](/images/2019-06-05/4.png)      

#### Instruction patching
REDQUEEN利用kvm和qemu的功能将比较指令替换为cmp al,al，同时用intel PT来去掉那些产生意外的情况（例如执行到一条指令的中间）。

#### Input validation and fixing
作者用下面的算法来修复输入中的比较指令。    
![avatar](/images/2019-06-05/5.png)      

### Evaluation
在这里作者基于3个问题进行evaluation。     
RQ1:基于input-to-state correspondence的技术能否在一系列不同集合的目标和环境上满足通用型要求？    
RQ2:基于input-to-state correspondence的技术和其他更复杂的方法，例如污点追踪，符号执行相比，有多大的优势？    
RQ3:以input-to-state correspondence为基础的技术在真实世界的fuzzing场景提供了什么样的改进？    
为此，作者首先在一系列不同的环境中、LAVA-M,CGC测试集上测试REDQUEEN的表现，接着在实际环境中运行，找到一系列真实软件的bug，最后比较了和其他kfal为基础的工具的性能。

#### Evaluation method
所有的实验基于一台运行着ubuntu 16.04服务器版本的机器，cpu是Intel i7-6700 processor (4 cores)，有24g内存。在测试中，所有的fuzzer工具被允许运行一个进程，基于未修改的binary统计code coverage。

#### LAVA-M
LAVA-M是一个专门的测试fuzzer的bug软件集合，它把常见的软件漏洞放在开源代码中fuzzer难以执行到的地方来测试fuzzer性能。作者将其他软件在这个测试集合上的结果和REDQUEEN上进行了对比，值得说明的是，由于有的fuzzer不开源的缘故，作者引用了别人的一些测试结果，不同的fuzz软件在测试时使用了不同性能的计算机，但大都跑了5个小时。     
![avatar](/images/2019-06-05/6.png)    
在LAVA测试集上，REDQUEEN找到了大部分列出的bug，甚至找到了一些被测试集认为使用fuzzer无法找到的bug，显而易见，REDQUEEN的性能超过别的fuzzer。

#### Cyber Grand Challenge
DARPA’s CGC是另一个常常使用的fuzzer测试集。同样的原因一些fuzzer的测试环境不同，在cgc中测试的binary个数也不同，最后作者只和VUzzer进行了对比，在31个binary上，作者在第一次运行REQUEEN上都找到了bug，而VUzzer只在25个上做到了。

#### Real World Cases
作者在binutils上进行了测试。由于发现bug的数量难以确定（多个crash可能是同一个bug），所以作者使用了code coverage作为衡量不同fuzzer发现潜藏很深的bug的能力。测试结果如下。    
![avatar](/images/2019-06-05/7.png)      
可以看出，无论在哪个binary，REDQUEEN的code coverage都是最大的。作者发现VUzzer严重依赖于一个假设，只有一种正确格式的输入，所以在binutils上运行效果不好。         
同样，作者使用找到了很多real world程序中的bug。    
![avatar](/images/2019-06-05/8.png) 

#### Basiline evaluation
作者在这里针对PNG format库lodepng进行测试，因为PNG格式又很多chunk链接，而每个都有一个crc32的checksum，对于很多fuzzer来说很难处理。
作者将kfal配置了专家库，不配置专家库和REDQUEEN进行对比，发现REDQUEEN最后能找到配置专家库所找到的的所有的code coverge。
![avatar](/images/2019-06-05/9.png)    

#### Performance
最后作者还对同样时间内获得新的code coverge所需要的新输入数目，和不同的fuzzer进行了比较，进一步说明REDQUEEN在性能上的优势。
![avatar](/images/2019-06-05/10.png)      

### Conclustion
在这篇文章中，作者提出了一种非常简单的提升fuzz效率的方法，基于一个简单的假设，即一个程序的输入很多时候被直接用来进行逻辑判断，所以在tracing中可以找到对应数据。虽然可能不能面对所有情况，但还是比较通用的，而且性能很好。