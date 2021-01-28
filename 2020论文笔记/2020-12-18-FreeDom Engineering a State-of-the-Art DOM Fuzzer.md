# FreeDom: Engineering a State-of-the-Art DOM Fuzzer

> 作者：Wen Xu, Soyeon Park, Taesoo Kim
> 
> 单位：Georgia Institute of Technology
> 
> 会议：CCS 2020
> 
> 论文链接：[FreeDom: Engineering a State-of-the-Art DOM Fuzzer@CCS'20](https://dl.acm.org/doi/pdf/10.1145/3372297.3423340)

## Abstract

Document Object Model (DOM) engine是现代浏览器的核心组件，负责在用户设备的交互式窗口中显示HTML文档，有着庞大的代码库和复杂的逻辑，是Web浏览器最大的错误源之一。实际应用中，对于DOM engine bug的主流检测技术是DOM fuzzing，但现有DOM fuzzer缺乏对数据依赖关系的刻画，而且无法进行coverage-guided mutation。

本文中，作者设计实现了FreeDom，也是第一个在分布式环境下同时支持document generation和coverage-guided mutation的DOM fuzzer。作者设计了FD-IR这一中间语言来描述HTML文档，从而保留了数据依赖关系和状态性。FreeDom从Safari、Chrome、Firefox这三个主流浏览器中发现了24个新的bug，其中被分配了10个CVE。性能上，与现有DOM fuzzer对比，FreeDom能够取得更高的吞吐、覆盖更多的代码快、触发更多的crash。

<!-- more -->

## 1 Introduction & Background

### 1.1 DOM & DOM Engine bugs

Web浏览器是以HTML文档为输入的，DOM就是一种HTML文档的一种描述。如图1所示，HTML文档包含initial DOM tree、CSS rules和event handlers三部分。

![](/images/2020-12-18/upload_3630c10a9ee13f6916b8313166b503b3.png)

- DOM tree：DOM在逻辑上将HTML文档视为树结构，由HTML文档文件指定初始对象树，每个节点代表要渲染的对象。
- CSS rules：用于指定呈现文档中的元素的样式。一条CSS rule包含一组CSS selector和一组CSS properties，前者指定元素，后者指定样式的属性。
- Event handlers：为了提供网页的交互性，DOM标准定义了在特定时间点或用户输入处触发的各种事件，可以在`<script>`中注册事件处理程序，从而以编程方式访问或控制树中的对象。

浏览器运行DOM引擎（如Apple Safari中的WebKit和Google Chrome中的Blink）来解释HTML文档。***作者针对的DOM bug是由DOM engine触发的内存错误（不关心UXSS等逻辑问题）***。这种客户端的bug会导致数据泄漏，甚至在渲染器进程的上下文中执行远程代码。尽管浏览器厂商通过各种方式来减少DOM engine bug，但是近年来仍有很多针对DOM错误的完整浏览器利用链

### 1.2 Existing DOM Fuzzers

DOM fuzzing作为主流的DOM engine bug检测技术，Microsoft、Google、Mozilla都有进行研究。表1列出了现有的DOM fuzzer，其中Domato是由Google Project Zero开发的，也被认为是最成功的一个，因此是作者本文比较的主要对象。

![](/images/2020-12-18/upload_70fa831ecf4d8c1998e0290a85491e89.png)

来看看当年Domato的测试结果，如下表（Safari重灾区）。实际上当时他们有进行实验性质的coverage-guided，并对IE浏览器进行了测试。不过可能是因为当时只能做到“添加”，所以并没有发现新的crash。

![](/images/2020-12-18/upload_4ccbbabefad5aaaf3d12d1e107e45aa5.png)

## 2 Motivation & Challenge

### Ineffectiveness of Static Grammars

对于这类复杂软件的fuzzing，非常关键的一点是避免语义错误。目前的DOM fuzzer使用***上下文无关的语法来***描述HTML文档。这种静态语法在很大程度上保证了生成的输入的句法正确性，但是无法描述整个输入中的每个数据依赖性。例如CSS selector、CSS property、Attribute name value和Attribute value这四种典型的上下文相关值就是难以用Domato的语法来正确描述。

### Lacking Coverage-guided DOM Fuzzing

正如表1中所示，目前缺乏mutation-based fuzzing技术在DOM fuzzer上的应用。这也正是受制于Domato等使用的静态语法，生成的测试用例都是stateless的，只能向已有的HTML文档中添加新数据，而无法进行数据的翻转、拼接等操作。作者这个工作也是第一次系统的研究coverage-guided mutation在DOM fuzzing中的应用，指出需要有状态的结构来表述而非简单的文本。

### Low-throughput executions

管理浏览器的lifetime是具有挑战的。不同于其他fuzzing的对象，除非发生崩溃，否则启动的浏览器实例永远不会自动终止。由于随时可能发生的动态渲染任务，也很难知道输入文档被完全处理的确切时间点。

现有DOM fuzzer采用的常见解决方案是在保守的时间限制下运行每个浏览器实例。例如，默认情况下，ClusterFuzz对每个输入最多测试10秒钟。当生成的输入具有一定的复杂性，并且可能花费大量时间进行渲染时是合理的。但mutation-based fuzzing处理的大多数输入较小，不需要那么长执行时间。实际上，对于Domato生成的60％以上的文档（通常大小超过200K字节），WebKit窗口将在1秒后完全变为空闲状态。因此这种固定时长会导致效率低下。

## 3 Design

为了解决上面说的这些挑战，作者设计了FreeDom，也是第一个在分布式环境下同时支持document generation和coverage-guided mutation的DOM fuzzer。FreeDom的设计如图3所示。

![](/images/2020-12-18/upload_d6f4433f1d8cd55a0a9a34065be13728.png)

### 3.1 Context-aware Documentation Representation

为了能够进行测试用例的mutation，作者设计了FD-IR这一中间语言来描述HTML文档。之前的DOM fuzzer生成的测试用例都是只有无状态的纯文本，导致无法进行突变。而借助FD-IR，作者实现了stateful、context-aware和extensible。具体来说有两类上下文：

- FD-IR维护了一个树状结构来保存全局上下文，包括初始DOM tree中的所有元素、标签和属性，还有所有token。这是前面说的四种数据依赖的基础。
- FD-IR还为每个JavaScript函数描述一个局部上下文，用于生成语义正确的DOM API调用。FD-IR知道定义局部对象的确切位置以支持各种API突变。

FD-IR的基础类是`Value`，包含`generate()`、`mutate()`和`lower()`这三个核心函数。

![](/images/2020-12-18/upload_25abfc69bbcb9a7ebfe01db9f24b8cd3.png)

基于FD-IR value，FreeDom能够进行DOM tree、CSS rule和event handler的生成、突变与合并：

- DOM tree：FD-IR采用多分支树，`<body>`是唯一的根节点，对每个节点，FD-IR记录类型、id、子节点列表和属性列表。
- CSS rules：FD-IR为每条规则记录一个selector列表和property列表（均为`Value`实例）
- Event handler：FD-IR维护了一个event handler的列表。除了main event handler，其他的handler总数是FreeDom预先定义的，不会在mutation过程中增加。每个event handler由一系列API调用以及局部上下文组成。涉及到的Value的generation和mutation都是在局部上下文上进行的，然后被简化为对应的JS代码单元。

### 3.2 Context-aware DOM Fuzzing

***Document Generation*** $FD_G$：

- 初始状态就是一个空的文档，仅包含一个`<body>`元素、一个空的main event handler和一个空的event handler列表。
- 首先生成DOM tree：重复调用$G_{t1}$方法插入一个元素、$G_{t2}$方法添加一个属性（基于全局上下文信息）、$G_{t3}$方法插入一个text节点（生成一个随机字符串）；
- 接着生成CSS rules：重复调用方法添加CSS selector、$G_{C2}$方法添加CSS property；
- 最后填充event handler：从全局和局部上下文中提取出可用的对象作为参数，随机把API调用加入到event handler，并记录返回的新对象。

每次突变后，central server会保存那些产生覆盖率提升的测试用例，这也提升了分布式环境下coverage-guided fuzzing的效率。另外，FreeDom在输入文档的处理大部分完成时就动态终止浏览器实例，也提升了吞吐率。

***Single Document Mutation*** $FD_M$：有了FD-IR表示的结构与提供的接口，变异就比较简单：

- Mutate an attribute value.
- Replace an attribute.
- Mutate a text node.
- Replace a CSS rule
- Mutate a CSS selector
- Mutate a CSS property
- Insert an API call
- Replace an API call
- Mutate API arguments

***Document Merging***：分四个步骤进行，分别是initial DOM trees（节点匹配后添加缺失的属性）、CSS rules（直接复制）、event handler merging（简单的shuffle，但不改变原序列中先后关系）以及references fixing。

![](/images/2020-12-18/upload_7f5dc532329b8f9400a7eee44dd47d94.png)


## 4 Evaluation

### 4.1 Discovering new DOM engine bugs

作者对HTML、CSS、SVG标准进行fuzz，并对不同OS平台的WebGL分别fuzz，断断续续跑了两个月，得到了表5的结果（10个CVE/24个bug）。这些bug都已经得到了浏览器厂商的确认。除了一些assertion、null dereferences和correctness issues，绝大多数错误都对安全相关的，帮助作者获得了10个CVE和65K美元的漏洞赏金。

![](/images/2020-12-18/upload_4e2d6279118afd93791577041eca8a13.png)

### 4.2 Effectiveness of Context-aware Fuzzing

与Dharma比较：覆盖率和crash均全面占优

与Domato比较：尽管覆盖率略低于Domato，但是能发现更多的crash。

作者也与现有的fuzzer进行了对比，来测试coverage-guided mutation的效果。结果显示，FreeDom能够取得更高的吞吐、覆盖更多的代码快、触发更多的crash，并能发现之前无法找到的复杂bug。

![](/images/2020-12-18/upload_c961392b853228a459cb16ea9b0a45d7.png)

![](/images/2020-12-18/upload_da0514847c8f0d147c19f07a9f568926.png)

![](/images/2020-12-18/upload_68f770517caa0609aca9680b6af78f6e.png)


### 4.3 Effectiveness of Coverage-driven Fuzzing

优势：

- $FD_M$的表现完全优于$FD_{M-}$(移除了merge）：多了1.8%的代码覆盖率和1.8倍的unique crash，其中两个是与heap-related memory的问题。相比之下，$FD_{M-}$无法找到任何与安全相关的crash。
- 与FDg相比，FDm-和FDm分别多访问约1.2%和2.62%的代码块。而且FDm成功地发现了三种新的crash，包括两种运行FDg 24小时都无法发现的安全相关的崩溃。

劣势：

代码覆盖率对于发现bug的影响是微弱的。与$FD_G$相比，$FD_M$的执行次数增加了3.5倍，覆盖范围提高了2.62%，但发现的unique crash少了3.8倍，错过了大约75%的唯一崩溃，包括$FD_G$在3个24小时运行期间发现的7个与安全相关的崩溃。

二者对比：

- 作为一个基于生成的模糊器，$FD_G$生成一个大尺寸的文档，其中包含大量随机选择的元素、属性和CSS规则，旨在在一次执行中涵盖各种DOM特性。
- $FD_M$关注的是其输入的重复突变和逐渐生长。与$FD_G$相比，更容易触发需要严格或微妙设置的崩溃。

![](/images/2020-12-18/upload_bff14b4919d383c3679083a3eb6bb7f2.png)

### 4.4 High-Throughput Executions

$FD_G$ 在原来的WebKit中总共发现了112个独特的崩溃，作者优化的WebKit中可以触发其中97%的崩溃，包括所有11个安全相关的崩溃。

![](/images/2020-12-18/upload_2751ad77be1fbbddd7e0e0142078e9cc.png)

> $FD_G$ test 26.92 documents/s > 硬编码5s时间 100核， 20ducuments/s

## Limitations：

1. FD-IR目前不支持直接将html文档转换成IR。但理论上是可以克服的，如果能解决这个问题，就能用现有的测试套件中大量语料库进行测试。
2. 其fuzzing本身的设计其实没有很出彩，缺乏比较好的调度策略等。