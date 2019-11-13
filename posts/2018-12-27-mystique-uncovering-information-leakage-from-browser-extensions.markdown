---
layout: post
title: "Mystique: Uncovering Information Leakage from Browser Extensions"
date: 2018-12-27 16:34:51 +0800
comments: true
categories: 
---


作者：[ Quan Chen](https://dl.acm.org/author_page.cfm?id=99659313998), [Alexandros Kapravelos](https://dl.acm.org/author_page.cfm?id=81436601692) 

单位：North Carolina State University

会议：CCS'18

原文链接：https://www.kapravelos.com/publications/mystique-CCS18.pdf


## 1 简介

浏览器的第三方扩展程序可以注入一些JavaScript代码得到一些私密信息（例如：URL、cookies、输入等），也可以使用特权JavaScript扩展API直接访问私密信息（例如 chrom.history）。

作者为浏览器扩展开发了一个污点分析框架Mystique，并用它来对浏览器Chrome扩展的隐私泄露问题进行大规模（181,683个扩展）研究。

首先作者提出了一种污点分析的hybrid方法：因为扩展的源代码可在JavaScript引擎运行，作者使用从JavaScript源码的**静态数据流和控制流**分析中收集的信息来实现和增强传统的污点分析，即将**动态污点跟踪和静态分析**相结合。基于此，作者进一步修改Chromium浏览器以支持对扩展的污点跟踪。

研究结果显示浏览器扩展对用户隐私构成威胁，并需要采取对策来防止行为不当的扩展滥用权限。

<!--more-->


对比之前的工作：

1. 信息流方面
    - 之前也有检测在Chrome上基于DOM的XSS攻击漏洞的污点分析工作，但是只处理了字符串之间的污点传播，不能分析扩展的隐私泄露问题。
    - 之前的工作有一些扩展SpiderMonkey JavaScript引擎在Firefox浏览器上的污点跟踪的实现，虽然可以对对象和控制流进行分析，但是所有的数据流分析都基于SpiderMonkey，不适用于Chrome。
    - 一些使用源码级重写的方法得到信息流，不能覆盖浏览器API和与DOM相关的信息流，或者只能提供比较粗糙的信息流。


2. 检测隐私泄露扩展
    - 之前也有工作做了扩展地隐私泄露的检测，但是他们的方法大多是监控和分析网络流量。



## 2 背景

### 2.1Chrome扩展框架

 内容脚本：可以动态注入和静态注入到Web页，需要在清单文件中申明。注入后与普通的JavaScript代码相同，可以访问DOM接口。

 背景页 ：一个扩展只有一个背景页，在生存时间里长时间运行管理状态；在申请权限后可以使用Chrome扩展API。

消息传递API：注册回调函数，可以在扩展内（几个脚本间）传递消息，也可以在扩展之间传递消息。

### 2.2 V8 JavaScript引擎

本文采用有2个编译器（Full-codegen和Crankshaft）的V8架构进行开发，在Full-codegen添加污点跟踪模块，但是在另一个引擎和最新的解释器-编译器对的架构下同样适用。



## 3 设计方案

### 3.1敏感数据源

- DOM接口
- Chrome扩展API

![img](/images/2018-12-27/table1.png) 



### 3.2 污点传播的静态分析

![img](/images/2018-12-27/figure2.png)

(1) (2) :  V8编译 JavaScript源码，Mysitique进行插桩，得到Native code。

（3）：在运行过程中，运行到某行，被插桩的Native code去Object Taint Table中寻找具体对象的污点状态，并对AST Taint Table进行更新。

- Object Taint Table：存储具体运行时JavaScript对象的污点状态
- AST Taint Table：存储污染的AST节点。每一个函数调用都有一个AST Taint Table。

（4）：在污点传播点，Native code触发Mystique进行污点传播（可能是递归的）

（5）：Mystique再次使用V8的基础结构生成函数的AST和DFG（Data Flow Graph），并存储。

（6）：Mystique根据DFG和AST Taint Table传播污点。

#### 3.2.1 运行时对象的污染表示

Object Taint Table是一个全局的hash表，存储运行时对象的污染状态，键值为对象的地址。

对于非堆分配变量，在Object Taint表中用等值的HeapNumber替换。

在V8的垃圾回收器工作时，更新对应的Object Taint表。

#### 3.2.2 AST结点的污染表示

AST Taint Table是一个二级查找表，键值为a)调用帧的指针 b)AST结点的内存地址。

由于大多数情况JavaScript函数并不会运行被污染的值，一般用不到a，得到第二级表后直接使用b进行查找。

#### 3.2.3 DFG生成

生成条件：变量、对象属性、函数调用。

RHS：赋值语句右边的表达式和控制结构的条件表达式。

 <img src="/images/2018-12-27/listing1.png" width="350"/><img src="/images/2018-12-27/figure3.png" width="350"/>

右图虚线箭头表示控制流，实线箭头表示数据流。

如果RHS中包括函数表达式，那么函数的返回值和进行递归调用中产生的流都要处理。

如果RHS中包括对象或数组，那么对象的所有属性和数组的每个元素都要处理。

#### 3.2.4 隐式数据流

此类的其他示例包括**函数返回值**和**复合类型**的表达式（例如，对象、数组）。对于前者，如果返回值表达式被污染，则返回值需要被污染；对于后者，如果指定对象中属性值的表达式被污染，则其对应的属性值也应该被污染（不会污染对象本身）。如第三行。

在图中以点-虚线的箭头进行表示。

#### 3.2.5 被污染的对象属性

如果对象的某个属性被污染，对象不会被污染；如果对象被污染，对象的所有属性都被污染。

#### 3.2.6 更新AST Taint表

插桩保证包含在DFG的AST结点被评估时一定会更新污染状态。

在Object Taint表中显示对象为污点则AST Taint表中对应AST节点标记为污点；在Object Taint表中显示对象不为污点则AST Taint表中对应AST节点做Untainting（可能在循环中出现）。

#### 3.2.7 污点传播点

为体现函数间的DFG需要对所有的函数调用进行**递归**处理污点传播。两点需要注意的地方：

1）在Untainting的操作下，为了解决循环中对象仅在最后一轮没有被污染的漏报，必须在基本块结束时触发污点传播。
2）在处理隐式数据流时也应该触发污点传播，隐式表达式也被视为RHS。

#### 3.2.8 处理污点字符串的eval

Cross-Site Scripting Prevention with Dynamic Data Tainting and Static Analysis.. In NDSS, Vol. 2007. 12.

如果JavaScript函数是从受污染的字符串编译的，则Mystique会污染该函数中赋值表达式的所有左侧目标；对于隐式流表达式，Mystique总是污染它们的值。



### 3.3 附加数据流路径

#### 3.3.1 DOM接口

 V8不实现DOM接口。相反，它是作为WebKit / Blink中JavaScript的附加组件实现的。DOM实现在V8外部，因此需要将写入DOM的JavaScript值转换为WebKit / Blink中的相应表示形式（反之亦然，从DOM读取值时）。

Mystique会在WebKit / Blink中污染在V8中受污染的值。

#### 3.3.2 Chrome扩展API

Chromium浏览器提供了的消息传递API，用于在网页内部执行的内容脚本之间传递消息，以及其余的扩展（例如，背景页）。此API允许JavaScript对象通过它传递。在内部，发送的对象首先被序列化（JSON.stringify），然后被提供给（C ++）IPC组件，以传递到消息管道的接收端，解析原始对象。但是，如果原始对象包含污染值，则解析回来的对象需要反映相同的污染状态。为了尽量减少对代码库的更改，作者选择不使IPC实现污染。相反，对于通过消息传递API发送的每个JavaScript对象，Mystique以递归方式访问其每个属性，并构造描述其污点状态的“元对象”。然后，该元对象也被字符串化，然后将结果预先添加到原始JavaScript对象的字符串化输出中。在接收端，元对象用于重建解析后的对象的污点状态。

扩展API引入的另一个数据流路径是chrome.tabs中的executeScript方法，它可以向网页中注入内容脚本执行。如果注入的脚本被污染，将以与污点字符串的eval相同的方式对待它们（参见第3.2.8节）。最后，Chromium提供了一个API（chrome.storage），允许扩展将JavaScript对象序列化为存储。与消息传递API类似，序列化使用JSON.stringify完成。因此，Mystique还通过使用与原始对象一起序列化的元对象来处理它的污点传播。



### 3.4 污点sink

- XMLHttpRequest：如果污点构成请求URL参数或请求正文的任何部分，则会引发警报。
- WebSocket：与XMLHttpRequest类似，如果通过此接口发送任何受污染的值，则会引发警报。
- chrome.storage：可能有扩展程序首先存储隐私敏感信息并且稍后仅批量处理它们的情况（例如，仅在收集阈值数量的URL访问之后）。（与所有动态分析系统一样，Mystique可能无法在运行时生成阈值数量的事件，以触发扩展的泄漏行为）
- 对于由扩展注入的DOM元素，如果其src属性包含污染值，则发出警报。对于这些元素，浏览器将尝试从src属性中指定的URL获取其内容。因此，扩展可以通过例如将其编码为src属性URL的一部分来泄漏敏感信息。



### 3.5 污点传播日志和sink报告

对于每个受污染的JavaScript对象，Mystique都会记录传播给它的JavaScript对象，以及JavaScript函数和函数中源代码的位置。对于作为污点源的JavaScript对象，其污点不会从另一个对象（例如document.location.href）传播。

如果污点传播是由元对象引起的（参见第3.3.2节），Mystique首先在文件系统上转储元对象描述的所有受污染的JavaScript值的传播日志。然后，传播日志的文件系统路径在元对象内编码。一旦原始对象被解析回来，这些文件系统路径将用于指示之前的污点传播。

当受污染的值到达污点时，Mystique：记录1）触发污点sink及其传播日志的污点值，2）当前JavaScript堆栈跟踪，3）沿传播路径的JavaScript函数的源代码（包括访问污点源的函数）



## 4 实现

### 4.1 将AST节点映射到JavaScript对象

建立一个映射表

### 4.2 优化污点传播

V8使用Context对象来模拟JavaScript中的全局变量作用域相对应的执行环境。为了最大限度地减少由于污点传播引起的运行时开销，Mystique原型实现中，仅对扩展的Context传播污点。

> 1. 给定全局变量范围的JavaScript代码无法访问其他代码中定义的对象。

> 2. 即使内容脚本是在context网页中注入和运行的，它们不能使用网页，其他内容脚本，甚至自己的扩展页定义的变量或函数；在内部，通过为内容脚本定义单独的Context对象来强制执行此规则。

> 3. 来自不同扩展的JavaScript代码被赋予单独的Context。

虽然Context类是在V8中定义和实现的，但它们的运行时实例化是由WebKit / Blink启动的，背景页和内容脚本的Context分别在main worlds 和 isolated worlds中运行，修改WebKit / Blink实例化时通知Mysitque。背景页的URL以“chrome-extension://”开头来标识。

对于非扩展Context的JavaScript代码不进行污点传播。

### 4.3 JSON.stringify 和 JSON.parse

如果传递给JSON.stringify的JavaScript对象受污染，那么输出字符串也需要被污染。

但是，如果输入对象本身没有被污染只包含受污染的属性，则污染不会传播到输出字符串。

为了传播污点，我们添加了可调用C++底层的JavaScript的trampoline函数。trampoline负责构造描述序列化对象污染状态的元对象。如果JSON.stringify的输入包含污染值，则trampoline函数将污染输出，并在传播日志更新元对象以待下一步的污染传播。

JSON.parse的输入字符串被污染，两种可能：

1）字符串在污点传播中的上一步是元对象，在这种情况下，Mystique根据元数据重建污点对象

2）非元对象，Mystique污染输出对象，并递归地污染其所有属性。

与JSON.stringify一样，使用trampoline函数来实现这些功能。

对JSON.stringify和JSON.parse的处理类似于我们处理Chrome消息传递和存储API的方式（第3.3.2节）。但JSON.stringify和JSON.parse的输出保留在V8堆内存中，而不是提供给C ++ IPC实现或外部存储。



### 4.4 JQuery请求协议

jQuery库经常被第三方Chrome扩展使用。 为了支持无协议的URL（以“//”开头），jQuery库首先通过读取DOM属性location.href来检索当前页面的URL并解析其协议（例如https），然后将其添加到请求之前。如果未指定协议，则为URL。然后将请求URL传递给XMLHttpRequest的open方法。 

由于location.href被Mystique视为污点源并且XMLHttpRequest被视为污点sink，因此无论原始请求URL是否被污染，都将触发警报。 为了解决这个误报，Mystique根据特定赋值表达式的AST结构创建了一个签名，该表达式负责错误地传播污点，并且在遇到表达式时不会传播污点。



## 5 评估

数据集：178,893（118,083个不同）的Chrome扩展程序和2,790个Opera扩展（Opera浏览器基于Chromium）

结果：Mystique标记了3,868个Chrome扩展和59个Opera扩展，可能会泄露隐私敏感信息。已标记的扩展名总数为3,868（2.13％）。

![img](/images/2018-12-27/table2.png)

### 5.1 测试内容

使用Selenium的ChromeDriver，驱动Chromium浏览器访问110个固定的URL。

模拟Web浏览的固定URL集分为两类：真实URL和模拟URL。

对于真实的网址，提供真实的网站回复；对于模拟URL，提供静态模拟页面。使用模拟页面的动机是通过避免花费在浏览器中加载实际页面所需的时间来缩短分析时间，同时仍然能够为正在分析的扩展生成许多URL加载事件。总共有10个真实的URL（流行的网站，如wikipedia.org）。模拟URL源自真实URL：对于每个真实URL，我们在浏览器中访问它并随机选择它链接到的10个URL。

### 5.2 结果分析

手动验证：检查污点sink是否包含敏感信息或从敏感信息获取的数据（包括加密、散列等）。

验证了Opera浏览器所有59个扩展和Chrome中随机选择369个扩展（95%的置信水平下5%的置信区间）。

确认了Chrome和Opera扩展分别有272和45个真（TP）、30和6个误报（FP）。

由于无法准确对污点Sink进行检查得到47个Chrome扩展和8个Opera扩展的真/假状态 ，对TP率进行了估算。

![img](/images/2018-12-27/table3.png)

### 5.3 案例分析

#### 5.3.1 受影响用户

确认泄露中用户数量前十的扩展。![img](/images/2018-12-27/table4.png)

#### 5.3.2 SimilarWeb Library和Web of Trust

Mystique总共在7000+个扩展中发现382个扩展（或99个不同的扩展，去掉版本差异）包括SimilarWeb库，之前的工作只能找到42个。并检测到了以下扩展：

 1）不直接打包SimilarWeb库代码，但是在运行时下载并执行它

2）使用库的缩小版本

Web of Trust（WOT）：根据热门评论为用户提供有关网站的声誉和安全信息的扩展。

Mystique能够将WOT扩展标记为泄露隐私敏感信息。通过检查污点传播报告，WOT扩展通过首先应用RC4加密然后用double-base64（即两次base64编码）对结果进行编码来模糊泄漏的数据。

#### 5.3.3 和网络流量分析的方法对比

以前使用动态分析识别隐私泄漏浏览器扩展的研究工作仅关注扩展产生的网络流量。由于他们缺乏对扩展内部详细数据流的深入了解，因此他们必须依靠启发式方法来尝试对任何编码参数进行去混淆并恢复明文。 

无法处理以下三种情况：

1）预先未预料到的编码方案

2）单向散列或加密

3）扩展在本地存储上保留隐私敏感信息 

#### 5.3.4泄漏到localhost的扩展

根据Mystique的分析结果，发现经常有一些扩展的污点sink对象似乎被发送到localhost的特定端口上。

这些扩展通常用作已安装在用户计算机上的本机应用程序的补充。应用程序负责监听localhost上的请求。

推断实际将浏览活动泄漏到远程服务器可能是应用程序。



## 6 不足

1. Mystique从根本上依赖于浏览器扩展的运行时动态分析，以便检测隐私侵入行为，所以与所有动态分析系统一样，恶意行为的成功检测取决于分析期间是否能触发此类行为。
2. 分析扩展的代码覆盖率受到限制。





