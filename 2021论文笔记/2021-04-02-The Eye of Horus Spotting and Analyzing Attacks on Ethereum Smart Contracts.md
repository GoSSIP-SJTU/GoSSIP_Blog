# The Eye of Horus: Spotting and Analyzing Attacks on Ethereum Smart Contracts

> 作者：Christof Ferreira Torres<sup>1</sup>, Antonio Iannillo<sup>1</sup>, Arthur Gervais<sup>2</sup>, and Radu State<sup>1</sup>
>
> 单位：<sup>1</sup>SnT, University of Luxembourg; <sup>2</sup>Imperial Collage London
>
> 出处：arXiv preprint 2021
>
> 原文：[The Eye of Horus: Spotting and Analyzing Attacks on Ethereum Smart Contracts](https://arxiv.org/pdf/2101.06204.pdf)

## 1 Abstract

在最近几年，以太坊受到了广泛的关注。其日交易量从2016年的10K，增加到了2020年的500K。进而以太坊智能合约（下称合约）也将与越来越多的资金流通挂钩。但在价值提升的同时也必然受到更多的攻击，并将造成大量的经济损失。目前为止，学术界和工业界也已经开发了大量的工具，在部署合约前检测其漏洞。但是这些工具中的大部分都仅仅检测合约存在的漏洞，没有检测其瘦到的攻击，更没有量化和追踪资产流失情况。

因此本文设计了HORUS框架，用于自动检测合约所受的攻击情况，并提供快速量化和追踪以太坊中资金流失情况的方法。在文中也对2020年5月前的合约进行分析测试。

<!-- more -->

## 2 Introduction & Background

区块链备受关注，合约安全关系着大量资金的安全。

截至2020年5月，以太坊市场价值为42，000，000，000 USD，其中最有价值的合约（WETH）中存有超过相当于2，000，000，000 USD的以太币。于此同时，以太坊的日交易量也大幅增长。因此这样庞大的价值意味着更多的攻击者会瞄准以太坊，攻击所带来的损失也将巨大。例如以前发生过的著名的TheDAO事件、以及两次Parity Wallet attack。

到目前为止的研究大部分集中在检测合约的漏洞上，忽略了对现实生活中合约遭受攻击的分析，更缺少了对攻击者行为和盗窃资金流动的分析。

虽然有少量的研究利用了历史交易信息来检测合约遭受的攻击情况。但是这些研究不是需要对以太坊客户端进行修改，就是需要编写很庞大且复杂的脚本来检测攻击，实用性不是很好。

**contribution**：
- 提出了HORUS工具，基于历史交易信息检测合约攻击。
- 利用此工具，分析出相应攻击下的资金流失量。
- 追踪资金流失链，用于攻击者行为分析。
- 对过去4.5年的合约进行了分析，找到了8095个攻击以及1888个存有漏洞的合约。
- 对最近出现的Uniswap攻击和Lendf.me攻击进行了分析。

## 3 The Horus Framework

Horus的整体框架如下图所示：
![Horus-Framework](/images/2021-04-02/Horus-Framework.png)

框架整体分为三个部分：extraction，analysis 和 tracing

- Extraction：输入要分析的交易列表，从中提取出执行的相关信息，存储在Datalog Facts中。
- Analysis：将Datalog 中数据的关系和查询请求作为输入，识别出针对这些Datalog 中数据的攻击。
- Tracing：将Analysis阶段得出的攻击者帐户列表作为此阶段输入，然后收集所有包含这些帐户的交易（普通交易、内部交易和代笔转移）。然后根据此可以得到一个可扩张的资金流图数据库。

### 3.1 Extraction

此阶段主要从以太坊客户端请求某一交易列表对应的执行信息，并将这些执行信息转换为能够反映其执行语义的逻辑关系。

> 执行信息：由执行的EVM指令列表组成，此指令列表中的记录包含了执行的opcode，程序计数器值，调用栈深度和当前栈的值。

然而，这样的执行信息是不能从区块链的历史数据中直接获取的，只能在合约执行的过程中记录下来。因此想要分析以往的交易执行信息，必须要重放以往的交易，记录执行过程中的相应信息。本文利用**Geth**所提供的`debug_traceTransaction`和`debug_traceBlcokByNumber`**RPC**接口来重放相应的交易和区块。

由于在执行信息中，有一部分的信息与本文的分析无关。正好**Geth**允许用户自己制定执行信息过滤器（JS语言），因此为了提高提取执行信息的效率，本文在不修改Geth的情况下，对**RPC**接口的依赖信息进行修改，通过减少过滤器的提取信息量来提升其效率。

> 这样的改进比直接修改Geth要好，因为修改RPC不会依赖于Geth的版本变更而受到直接影响。

本文制定的过滤器，去除了原先执行信息所包含的当前程序计数器值，剩余gas量和指令的gas花费。除此之外，针对每个执行的指令只提取相应的栈元素和内存片段，而不会再返回整个栈和内存信息。

![List of Datalog facts](/images/2021-04-02/List of Datalog facts.png)


上图展示了利用本文所设计的提取器，在针对执行信息的每一条记录能够获取得到的内容列表（Datalog facts）。

**Dynamic Taint Analysis**

本文的提取器利用了动态污点分析思想来追踪指令间的数据流，记录在**data_flow**中。当污点从源头被引入的时候，安全人员可以通过**data_flow**字段来审计这些数据是否流入了敏感指令中。

> 源头指该指令会引入不可信任的数据，例如  CALLDATALOAD或CALLDATACOPY
>
> 敏感指令例如 CALL或SSTORE

本文设计的动态污点分析引擎将会分析每条执行的指令是否为**源头**。在遇到**源头**时，给相应的栈值（或者memory区域和storage位置）做上标记。下图表示了污点传播的过程。

![propagation of taint](/images/2021-04-02/propagation of taint.png)

**Execution Order**

考虑到存在例如Parity wallets hacks这样利用两个交易的执行顺序漏洞的攻击，本文用一个三元组o = (b, t, s)来定义一条指令的执行顺序。其中，b表示区块号，t表示交易号，s表示执行步骤。执行步骤是一个简单的计数器，在每一个交易开始的时候都会重置，在每一条指令执行之后则会自增。这样就能准确定确定出每条指令的执行顺序。

### 3.2 Analysis

本文根据制定好的相应攻击下Datalog所呈现的特征，制定Datalog relations 和 Datalog queries列表。利用Souffle工具作为Datalog分析引擎，匹配Datalog facts 与所制定的列表，以此来定位攻击信息。

本文提供了reentrancy，Parity wallet hacks, integer overflows, unhandled exceptions 和 short address attacks的Datalog queries列表。

**reentrancy**

如下图所示，本文通过查找是否有同一个caller在调用同一个callee来判断reentrancy的存在与否。

![reentrancy](/images/2021-04-02/reentrancy.png)

即通过检查两个成功的**call**操作，是否有相同的**hash**，**caller**，**callee**，**id**和**branch**的值，并且第二个**call**的**depth**的值高于第一个。

然后再检查是否存在两个相同的**storage**操作，而且这两个操作的**depth**与第一个**call**相同。同时这两个操作中，第一个是SLOAD指令，第二个是SSTORE指令。

**Parity Wallet Hacks**

总共分为两种攻击类型

- 第一种：首先检查是否存在两个**transaction**字段，记为t1和t2，且这两个交易都有相同的发送者和接收者。其中，t1的**input**值的前4字节代表了函数initWaller函数的起签名（e46dcfeb），并且t2的**input**前4字节与execute函数的签名（b61d27f6）匹配。然后再检查是否出现**call**字段信息，并且这个**call**字段是t2交易中的一部分，t2在t1交易之后（即，block1 < block2 或者 block1 = block2, index1 < index2）。具体如下图所示：![Parity Wallet Hacks1](/images/2021-04-02/Parity Wallet Hacks1.png)

- 第二种：这种类型的检查与第一种非常类似，只是在这里需要检查t2交易的input值是否与kill函数的签名(cbf0b0c0)匹配，并且t2交易中包含**selfdestruct**（非**call**）字段信息。具体如下图所示：![Parity Wallet Hack2](/images/2021-04-02/Parity Wallet Hack2.png)

**Integer Overflow**

如下图所示，首先检查是否有数据从**CALLDATALOAD**或**CALLDATACOPY** 中流入**arithmetic**操作中，其中**arithmetic_res**的值和**evm_res**不匹配。然后检查**arithmetic**的数据是否流入了调用SSTORE指令的**storage**操作中，并进行了**erc20_transfer**操作，其中**arithmetic**的**amount**字段是算术运算的操作数之一。满足上述情况则为整数溢出。

![Integer Overflow](/images/2021-04-02/Integer Overflow.png)


> 注：本文的整数溢出的检查只检查针对ERC_20代币的部分。

**Unhandled Exception**

如下图所示，首先检查**call**字段执行CALL指令时失败了（执行的结果是0），同时**amount**的值大于0。然后查看执行的结果没有在**condition**字段中，则说明出现未处理异常漏洞。

![Unhandled Exception](/images/2021-04-02/Unhandled Exception.png)

**Short Address**

如下图所示，在检测短地址攻击的时候，首先要检查**transaction**字段信息中**input**值的前4个字节是否为transfer函数或者transferFrom函数的函数指纹（a9059cbb和23b872dd）。然后，若匹配到transfer函数，则检查**input**的长度是否小于68字节（4字节(函数指纹) + 32字节(目的地址) + 32字节(转移金额)), 若匹配到transferFrom函数，则检查**input**的长度是否小于100字节（4字节(函数指纹) + 32字节(源地址) + 32字节(目的地址) + 32字节(转移金额))。最后再检查是否出现了**erc_transfer**字段信息。

![Short Address](/images/2021-04-02/Short Address.png)

### 3.3 Tracing

Horus框架的最后一部分是最总资金流流失情况。在分析阶段得出的恶意交易信息后，追踪器首先要提取出在这些交易中发送者的地址和时间戳信息，其中发送者地址就被认为是攻击者的地址。然后追踪器利用Etherscan的API接口，获得每个发送者地址的正常交易、内部交易、代币转换信息，并将这些信息加入Neo4j graph数据库。所有的帐户都描述为顶点，交易作为相应顶点的边。

对于一个帐户的交易，载入不超过1000个，为了避免数据库充斥着 mixing services, exchanges 和 gambling smart contracts的信息而过于庞大。

显然本文的追踪不能覆盖全部的节点，因为mixing services, exchanges等阻碍了更进一步的溯源。但是，本文的结果仍然足够判断攻击者资金的流动情况。

## 4 Evaluation

**实验设备**：
- 64GB内存
- Intel(R) Core(TM) i7-8700 CPU  3.2GHz  12核 
- 64-bit Ubuntu 18.04.5 LTS
- Geth v 1.9.9
- Souffle v 1.7.1
- Neo4j v 4.0.3

**实验内容与结果**：

本文选取了几个在以太坊比较常见的合约攻击形式，利用此工具对收集得到的1,234,197 个合约组成的371,419,070个交易进行了分析，得到下图的结果。可以看到此工具的误报率非常的低，给出的结果非常可信。同时，本文也将此结果与目前为止一些不错的工作进行了比较，效果也都有所提升。

![evaluation](/images/2021-04-02/evaluation.png)

同时本文基于工具分析的结果，分别得出了合约攻击数量与合约部署数量的趋势对比图和各类常见合约攻击的出现频率图。（如上fig. 3和fig. 4所示）可以发现随着时间的推移，攻击的数量确实有在减少，合约看起来更加的安全了。但是也可看到，注入重入攻击和未处理异常返回值攻击这类的经典攻击依然存在，带来的资金损失依旧非常大。

![合约部署数量及攻击数量对比图](/images/2021-04-02/合约部署数量及攻击数量对比图.png)

![合约各种攻击频率统计](/images/2021-04-02/合约各种攻击频率统计.png)

在文章的最后，作者也用Horus对最近比较重要的两个攻击（Uniswap和Lendf.me）进行了详细的分析，得到如下两张图对损失资金分析的结果。

![Uniswap](/images/2021-04-02/Uniswap.png)

![Lendf-me](/images/2021-04-02/Lendf-me.png)

## 5 Conclusion

总的来说，本文提出并实现的Horus工具检测并分析了针对合约的攻击行为，同时给出了资金损失情况的分析和统计，并给出与攻击相关的资金流图。除此之外，本文根据此工具得出的结果也揭示了合约安全问题依旧严重。在目前以太坊系统中，虽然攻击的数量看起来随着安全检测工具的发展有所减少了，但是诸如未处理异常攻击及重入攻击仍然存在，对以太坊的威胁依然很大。