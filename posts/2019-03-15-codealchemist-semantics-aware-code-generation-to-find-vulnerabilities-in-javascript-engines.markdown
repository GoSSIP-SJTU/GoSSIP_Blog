---
layout: post
title: "CodeAlchemist: Semantics-Aware Code Generation to Find Vulnerabilities in JavaScript Engines"
date: 2019-03-15 15:04:40 +0800
comments: true
categories: 
---

作者：HyungSeok Han，DongHyeon Oh，Sang Kil Cha  

单位：kaist    

出处：NDSS 2019    

原文：[https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_05A-5_Han_paper.pdf](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_05A-5_Han_paper.pdf)

<hr/>
## 1.背景
Javascript引擎已经成为现代浏览器为网站提供动态和可交互性的核心部分。截止2018年7月，大约有94.9%的网站（链接）使用javascript。JS的流行令浏览器的JS引擎成为了一个热门的攻击目标。目前，已经存在大量很多针对JS引擎漏洞的fuzzer。例如[LangFuzz](https://www.usenix.org/conference/usenixsecurity12/technical-sessions/presentation/holler),[jsfunfuzz](https://github.com/MozillaSecurity/funfuzz),[skyfire](http://www.ieee-security.org/TC/SP2017/papers/42.pdf),[TreeFuzz](http://mp.binaervarianz.de/TreeFuzz_TR_Nov2016.pdf)。这些工具都存在着一个共同的缺点：生成的语法正确的测试用例大多数在语义上都是有错误的。

<!--more-->

## 2.motivation
作者的研究源于在jsfunfuzz上进行的一项研究。作者使用jsfunfuzz在4种主流的JS引擎（ChakraCore, V8, JavaScriptCore, SpiderMonkey）上进行了测试，并且发现了下面有趣的现象：1.所有jsfunfuzz生成的js文件都产生运行时错误。2.每个文件都只在处理了几条语句后就抛出了运行时错误。事实上，99.5%的js文件运行了不到3条语句。    
![alt](/images/2019-03-15/1.png)
## 3.提出的方法和解决的问题
目前的js引擎fuzz方法，大多使用两种方法，1.将少量JS代码文件作为seed，将它们分割为一定代码片段并重新组合;2.使用一定的语法生成测试用的js语句。然而前者在执行中可能引用未定义的变量，后者则无法生成语义上有效的测试用例。    
基于上面的方法对语义有效性判断上的缺陷，作者提出了新的fuzz用例生成方法。这种方法将seed测试文件分为具有组合限制（assembly constraint）的代码块（code bricks），合并时添加在尾部的代码块中所有变量必须在之前的代码块中定义并有符合的类型。相比于其他最新的JS引擎fuzzer，这一感知语义的代码组合技术可以显著的减少生成的测试用例在fuzz时的运行时错误。同时，能够自然的从seeds中学习语义信息，不需要人工实现一个语法器。作者实现了一个原型CodeAlchemist。
## 4.技术方案
这里首先介绍作者提出的语义感知的代码块组合技术（semantic-aware assembly），接着介绍作者实现的原型CodeAlchemist的架构。
### 4.1. semantic-aware assembly
CodeAlchemist要解决的第一个问题就是生成语义有效的测试用例。使用的semantic-aware assembly方法第一步是将seeds分为可以重新组合的代码块。这里，一个代码块被定义为一个有效的JS抽象语法树。一个简单语句或一个块语句都可以成为一个代码块。    

每个代码块都被标记了一个组合限制条件。它包含先决条件(precondition)和后向条件(postcondition)两个部分。前者是一组变量集合和它们的类型；后者则表达了在这个代码块之后，什么类型的变量是可以使用的。

代码组合时，前面的代码块的postcondition必须满足当前代码的precondition。    
![alt](/images/2019-03-15/2.png)
### 4.2. CodeAlchemist Arch
如图，CodeAlchemist可以分为*SEED PARSER, CONSTRAINT ANALYZER*和*ENGINE FUZZER*三个部分。

![alt](/images/2019-03-15/3.png)
#### seed parser
seed parser将输入的JS文件seeds分解为代码块集合。    

首先按照js语句的粒度将seed文件分解，同时，一个块语句会生成多个变体来增加代码块的丰富性。例如，`while (x) { i += 1; }`会产生`while (s0) { s1 += 1; }`,`while (s0) {}`,`{ s0 += 1; }`多个代码块，通过组合可以产生嵌套循环等结构。    
所有代码块的集合形成一个代码块池。部分函数会产生非预期的的结果。例如，如果代码块包含eval函数或Spidermonkey的crash函数，则从代码块池中删除。
#### constraint analyzer
constraint analyzer用插桩技术为每个代码块推断并标注组合时的限制条件。    

首先用经典的数据流分析use-define chain技术确定需要定义的变量和新定义的变量。    
由于JavaScript的类型是动态的，变量类型可以在执行过程中改变，CodeAlchemist使用在运行时插桩的方法确定变量的类型。    
为代码块中的变量进行保留语义的重命名。


#### engine fuzzer
engine fuzzer从代码块池中随机选择代码块组合，生成测试用例并交给JS engine执行。

使用的算法如下图所示，有代码生成的最大循环。重新构造代码块概率，语句块内最大语句数，最大嵌套层数4个参数作为参数输入。 
    
![alt](/images/2019-03-15/4.png) 

算法在一个循环中按照某个概率p从代码块池中选择代码块，或构造新的代码块添加到生成的测试用例尾部。    
在重新构造代码块时（GenBlkBrick），这个算法首先选择按照某个概率p，递归的选择代码块（PickBrick）填充到当前代码块的空语句块内或继续构造代码块（GenBlkBrick）填充到当前的空语句块内。同时，随机删去当前代码块的部分postcondition（GetDummyBrick）来减少可以选择的代码块的数量。


## 5.实现
作者使用了600行的Javascript代码，100行的c代码和5000行的f#代码来实现CodeAlchemist。
## 6.评价
### 测试设备
作者在一台配备两个Intel E5-2699 v4 (2.2 GHz) CPUs (88个核心)和512 GB内存，运行64-bit Ubuntu 18.04 LTS系统的计算机上进行测试。
### 测试集
使用的浏览器为截止July 10th, 2018的最新版本，包括(1) ChakraCore 1.10.1; (2) V8 6.7.288.46; (3) JavaScriptCore 2.20.3; (4) SpiderMonkey 61.0.1。 

seeds文件来源于4个主要的JS引擎的测试文件和[Test262](https://github.com/tc39/test262)的测试代码。最终收集到63,523个独立的js文件，另外还有169个poc相关的js文件共63,692个文件。


### Questions
作者通过测试衡量以下问题：

* CodeAlchemist能否生成语法正确的测试集？
* fuzzing中使用的参数是否会影响CodeAlchemist的有效性，如果是这样，应该选用什么参数值？
* 在寻找bug方面，CodeAlchemist和相比怎么样？
* CodeAlchemist能否发现之前未发现的bug？


### Q&answer1
CodeAlchemist能否生成语法正确的测试集？    
作者使用CodeAlchemist生成的测试用例在ChakraCore进行测试，对生成的运行时错误的数量进行衡量。    
衡量时的判断指标为N语句成功率，即运行前N句语句时的成功率。与jsfunfuzz进行比较。    
![alt](/images/2019-03-15/5.png)
发现CodeAlchemist相比jsfunfuzz生成了平均6.8倍的有效测试用例。
### Q&answer2
fuzzing中使用的参数是否会影响CodeAlchemist的有效性，如果是这样，应该选用什么参数值？    
为了理解不同参数值带来的影响，作者针对两个参数进行实验。 (1) 组合语句块的最大循环次数如何影响生成用例的有效性；(2)重新构造新语句块的概率p如何影响bug的搜寻能力。 
#### 组合语句块的最大循环次数i_max
直觉上来说,i\_max越大，生成的代码文件具有更为复杂的结构和更多的语句，令CodeAlchemist分析不精准。    
对此，作者使用CodeAlchemist对i_max从1到20，生成每个i\_max的100,000个测试用例。并把从新构造语句块的概率p定为0，减少一定的随机性。    
![alt](/images/2019-03-15/6.png)    
结果表明，有效的语句数向8收敛。i\_max使用8比较合适。
#### 重新构造新语句块的概率p
通常认为，JS引擎的bug通常发生在处理高度结构化的js代码的时候。因此，通过改变重新构造的概率p，可能会对发现的bug数量产生影响。    
作者同样在ChakraCore 1.10.1进行测试，将p从0%变化到64%，测试结果如下。        
![alt](/images/2019-03-15/7.png)       
结果表明，重新构造新的语句块显然有助于发现bug，在p大于0时，CodeAlchemist至少发现了3个bug。但进一步提高p并不一定会提高发现的独立的bug数量。设置p为0.16是一个较好的值。
### Q&answer3
在寻找bug方面，CodeAlchemist和其他流行的fuzzer相比怎么样？    
作者将CodeAlchemist和jsfunfuzz，IFuzzer（Langfuzz的一个变种）进行了对比。CodeAlchemist使用之前提到的i\_max=8 和 p=0.16参数。（2112 cpu hours)
    
* 在旧的ChakraCore上的测试对比。使用2018年1月1日时最新的 ChakraCore。        
![alt](/images/2019-03-15/8.png)          
CodeAlchemist找到了最多的bug数，是jsfunfuzz的两倍多。
* 在最新的（2018.7.10）JS引擎上的对比。        
![alt](/images/2019-03-15/9.png)       
显然CodeAlchemist对新的和旧的浏览器，fuzz找到bug的能力都超越了目前其他的fuzzer。

### Q&answer4 
CodeAlchemist能否发现之前未发现的bug？    
作者对CodeAlchemist遇到的crush进行分析，发现了4种JS引擎有下表中的19个bug，其中有11个是可以利用的。8个是已知的，11个是可以先前未发现的。    
![alt](/images/2019-03-15/10.png) 

## 7.总结和看法
作者实现了CodeAlchemist,第一个能够针对JS引擎生成语义有效的测试用例的fuzzing系统。CodeAlchemist从一系列JS seed文件中学习语言语义，并生成一个语句块池供组合并生成新的测试用例。     
作者使用的方法比较简单，可以说是对Langfuzz方法的一些改进，得出的结果也比较好。相比于之前的方法，从结果来看，比较重要的是CodeAlchemist能够利用包含块语句的代码块进行符合语义的重新构造组合，可以增加很多之前完全生成不了的测试用例。

   
