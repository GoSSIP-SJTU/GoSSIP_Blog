---
layout: post
title: "RWGuard: A Real-Time Detection System Against Cryptographic Ransomware"
date: 2019-04-28 13:58:16 +0800
comments: true
categories: 
---

作者：Shagufta Mehnaz, Anand Mudgerikar, and Elisa Bertino

单位：Purdue University, West Lafayette, IN, USA

出处：International Symposium on Research in Attacks, Intrusions and Defenses (*RAID 2018*)

原文：https://link.springer.com/chapter/10.1007/978-3-030-00470-5_6

---

本文介绍了一个启发式的勒索软件检测工具RWGuard。RWGuard通过实时地记录进程对文件的操作，对这些操作进行量化考核从而识别出勒索软件进程。

#### 背景介绍

勒索软件是一类恶意软件，这类软件通过加密受害者机器上的文件进而向受害者勒索赎金来换取解密密钥。通常这类软件会遍历主机上的所有文件，对每个文件生成单一密钥并且使用这个密钥对该文件进行加密，然后使用恶意软件制作者的公钥进行加密密钥并保存在受害机器本地。所谓勒索受害者换取赎金指的是，在受害者支付赎金后，恶意软件制作者使用私钥对这些密钥进行解密。

<!--more-->
#### RWGuard设计

![](/images/2019-04-28/1.png)

RWGuard在各个目录中生成了诱饵文件，正常情况下，该诱饵文件是不会被写入的，如果被写入，则说明受到勒索软件攻击，则写入操作所在的进程被识别为恶意进程。

如上图所示，RWGuard开发了一个内核模块，该模块利用windows提供的minifilter过滤框架对文件系统的操作进行记录，并将该记录输入到用户层的处理组件中。DMon模块负责对诱饵文件操作的处理，如果检测到有进程对这些文件进行写入操作，则将该进程标记为恶意进程并终止该进程。如果写入的文件不是诱饵文件，则交由其他模块处理。

##### PMon

![](/images/2019-04-28/2.png)

FCMon监控系统中所有进程的文件操作，根据文件操作来判断是否为恶意进程，FCMon所考虑的文件操作如上图所示。主要包括为文件打开，写入与关闭等。第11个衡量标准，临时文件，通常被勒索软件用来生成原始文件的备份来加密。

RWGuard在FCMon中并不是使用简单的数值对比来确定结果，而是使用机器学习的方法，将这11个维度作为参考参入，训练出一个能根据不同的维度值来判断的模型。为了训练这个模型，作者选取了Wannacry, Cerber, CryptoLocker, Petya, Mamba, TeslaCrypt, CryptoWall, Locky, and Jigsaw作为恶意软件标本，并使用 Explorer.exe, WmiPrvSE.exe, svchost.exe, FileSpy.exe, vmtoolsd.exe, csrss.exe, System, SearchFilter-
Host.exe, SearchProtocolHost.exe, SearchIndexer.exe, chrome.exe, GoogleUp-
date.exe, services.exe, audiodg.exe, WinRAR.exe, taskhost.exe, drpbx.exe,
lsass.exe等作为正常的软件标本。将恶意软件与正常软件的文件操作结果进行手工标记，然后使用机器学习进行训练，最后得出一个可以根据输入来确定是否为恶意进程的模型。

##### FCMon

由于某些正常的软件如vmware, chrome也会有与勒索软件类似的文件操作，因此作者将写入的文件内容也考虑进来。即：写入前后的文件相似性，熵值的对比，文件类型是否改变，文件大小是否被改。

RWGuard的衡量标准为，如果文件类型被改变，或者达到任意另外三个要求其中之一的阈值，则认为是一个加密行为。

##### FCls

RWGuard的FCls模块作用为，当一个文件操作被FCMon和PMon认定为加密时，确定该加密行为是由于正常的用户加密行为导致的，还是勒索软件导致的。

RWGuard的衡量标准为，通过对正常的用户加密行为建模来确定该行为是否为恶意的加密。如果，在原始文件目录中对该文件进行加密，并且不改变文件类型，则这个行为很大程度上是非恶意的。

##### CFHk

![](/images/2019-04-28/3.png)

RWGuard的CFHk模块对windows的密码操作接口进行hook，被hook的函数如上表所示。CFHk模块记录被加密的文件路径，文件名，以及用到的密钥，加密模式等等。如果被确定为恶意软件加密，则该文件可以使用这些信息进行恢复。

但是我在这里无法知道CFHk模块是如何得知被加密的文件名以及路径，因为被hook的函数并没有提供类似的参数。这里作者没有表达清楚。并且使用windows密码操作接口的也仅仅是一小部分，所以十分有局限性。

#### 评估

##### 勒索软件选取

Locky, Cerber, Wannacry, Jigsaw, Cryptolocker, Mamba, Teslacrypt, Cryptowall, Petya, Vipasana, Satana,
Radamant, Rex, and Matsnu

##### 检测效率

![](/images/2019-04-28/4.png)

上图展示了在有诱饵文件与没有诱饵文件时，RWGuard在检测到勒索软件进程时，已经发现的write,read,open,close的文件操作请求数量。可以观察到在有诱饵文件时，文件写入操作(第一个图)的请求数量是小于10的，可以观察到RWGuard可以很有效的检测到勒索软件。

![]( /images/2019-04-28/5.png)

上图所示为RWGuard在检测到勒索软件前勒索软件运行的时间，RWGuard在平均小于8S的时间内检测出来了勒索软件。

##### 性能开销

磁盘开销：由诱饵文件带来的磁盘开销平均为5%

![](/images/2019-04-28/6.png)

内存开销：内存开销主要由于处理文件操作请求的log造成，如在计算文件操作前后的相似度，熵值比值等。如上表所示。

时间开销：时间开销主要由hook造成，而这里仅仅hook了几个密码库函数，并且这些函数并非常用函数，所以带的开销在10ms以内，可以忽略不计。

#### 评价

这篇论文有以下几个缺点：

1. 没有讲述好几个模块之间的关系，也就是说，如果在被操作的文件不是诱饵文件时，判断是否为勒索软件加密是否需要同时满足这四个模块的要求或者满足其中一个条件即

2. 处理文件操作log文件时，多久的判定结果可以作为最终结果，作者没有给出。如果一个恶意进程首先进行了长达一个小时的准备工作，那么该进程在前一个小时没有文件操作，那么该进程是否被判定为勒索进程？

3. 实验设计逻辑不太好，实验部分虽然已经把大部分需要展示内容展示出来，但是整体逻辑不够通顺。

4. 对诱饵文件，笔者的考虑是，如果加密操作对象为诱饵文件则判定为勒索软件加密，那么仅仅依靠这一个标准是否已经足够作为依据来判断是否为勒索软件。
5. 不够新颖，此前已经有论文讲过使用启发式的方法来解决勒索软件检测问题，并且对于由于检测延迟带来的文件被加密而不可挽回，shadow file是一个更为可行的办法，在这篇论文里对于这种由于轻微的检测延迟带来的文件被加密并且没有使用标准的windows加密库的勒索软件是没有处理办法。



