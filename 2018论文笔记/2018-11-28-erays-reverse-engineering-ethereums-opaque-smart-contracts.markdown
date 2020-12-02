---
layout: post
title: "Erays: Reverse Engineering Ethereum's Opaque Smart Contracts"
date: 2018-11-28 22:55:59 +0800
comments: true
categories:
---


作者： Yi Zhou, Deepak Kumar, Surya Bakshi, Joshua Mason, Andrew Miller, and Michael Bailey

单位：University of Illinois, Urbana-Champaign

出处：USENIX Security '18

原文：https://www.usenix.org/conference/usenixsecurity18/presentation/zhou

作者开发了一个针对以太坊智能合约的逆向分析工具Erays，用以将bytecode翻译成便于人类分析的伪代码。并且使用这个工具分析了以太坊生态系统的代码复杂性和重用性，以及将闭源的代码和开源的代码比较，降低整个生态系统的隐匿度。最后，作者给出使用Erays逆向4个智能合约的例子。

-------------------

## Background
以太坊作为第二代电子加密货币，其中的流通金额超过$10B USD。但是，以太坊智能合约容易出错，并且含有较高的金融风险。截止2018年1月份，作者采集了区块链上的所有智能合约，其中包含34K的独立合约。在这34K的智能合约中，26K(77.3%)没有源代码，并且涉及了31.6%的交易和$10B USD的金额流通。

<!--more-->
## Opacity in Smart Contracts
这一部分主要采集了闭源的智能合约，并且分析了这部分合约在整个以太坊中的占比和重要性。

采集的结果如下。
![table1](/images/2018-11-28/table1.png)
为更好的量化每个合约的重要性，作者采取了两个指标。
- 合约存储的金额
- 合约的交易数量

结果显示，不论看哪个指标，闭源的合约都在整个以太坊中占小部分。尽管如此，这些闭源的合约还是占有数量可观的份额。
<!--more-->
## System Design
Erays的输入为十六进制的字节码（bytecode），输出为易于人类阅读的伪代码。这一部分主要内容为Erays执行的核心步骤。
### 1.Disassembly and Basic Block Identiﬁcation
将bytecode线性扫描，翻译成EVM指令。在将这些指令以jump和jumpdest为标识，分块整合。

### 2.Control Flow Graph Recovery

生成控制流图主要在于找到给定block的后继。那么对一个block来说，只要查看最后一条指令即可。有三种情况：

1. 没有改变控制流

2. 程序停止指令（STOP，REVERT，etc.）

3. jump和jumpi

和基于寄存器的指令不同，EVM没有寄存器，指令的操作数全部存储在栈上，指令执行的时候直接从栈上取操作数。这造成了CFG恢复的困难，因为不能从指令中直接知道跳转的地点。因此，作者给出了解决该问题的算法，如下。

```
explore(block, stack):
	if stack seen at block:
		return
    mark stack as seen at block
    	
	for instruction in block:
		update stack with instruction save stack state
	save stack state
	
	if block ends with jump:
        successor_block = stack.resolve_jump 
        add successor_block to CFG 		
        explore(successor_block, stack) 
    if block falls to subsequent_block:
		revert stack state 
		add subsequent_block to CFG 
		explore(subsequent_block, stack)
```

算法中先检查了block之前有没有来过，有的话直接return了。这里能这么操作一是让递归能回退，二是因为虽然block可以是很多jump/jumpi的jumpdest，每次进来的栈可能都不太一样，但是对生成**后继**的block没有影响。

### 3.Lifting

这一部分将EVM基于栈的指令提升为基于寄存器的指令。这么做主要是因为转化后的指令不需要跟踪栈的状态，便于下一步的优化和转化为伪代码。为了让代码更加简洁和可读，作者还引入了一些新的指令。其中需要注意的是*INTCALL*和*INTRET*。这两个指令用来调用和返回内联（internal）函数，即仅仅使用无条件转移jump的函数。这两个指令的引入能让指令代码贴近高级语言的代码，让后面的转换更加简单便捷。

为了消除栈，作者做了一个映射，将栈转化为寄存器组。操作很简单，从栈底到栈顶，栈的每个item用一个寄存器代替，编号。下面是例子。

![lifting](/images/2018-11-28/lifting.png)

### 4.Optimization

之后作者做了优化，优化的算法在references里面。优化结果如下图。

![optimizing](/images/2018-11-28/optimizing.png)

### 5.Aggregation

这一部分将指令压缩，用到的方法有如下。

1. 将指令转化为三地址形式的**表达式**
2. 替代

转化为表达式的例子如下。

```
SLOAD $s3, [0x1] 										$s3 = S[0x1] 
GEQ $s3, $s2, $s3 										$s3 = $s2 ≥ $s3
JUMPI 0x65, $s3											if ($s3) goto 0x65
```

表达式能够更加接近高级语言。

替代就是将下文用到的内存和寄存器符号用上文的具体的值去替代。比如说，某处有语句

```
$r = RHS
```

那么，在可行的情况下，将下文全部的$r 替换成 RHS。具体来说，以上面的三行代码举例，经过替代后，就可以变成：

```
if ($s2 ≥ S[0x1]) goto 0x65
```

这种简单的形式。

### 6.Control Flow Structure Recovery

这部分进一步压缩控制流，把go to语句转换成for循环和while循环。

### 7.Validation

这一部分检验一下最终的代码和最原始的代码是否在业务和逻辑上有效且一致。

最终结果为，Erays没有通过3.22%的测试集。

### 8.Limitation

主要限制在于输出代码的可读性，以及对复杂数据结构（如mapping）的一些操作。并且，Erays没有对类型和变量做恢复。

> 上面工具的逆向过程能给手工逆向一点启发

## Measuring Opaque Smart Contracts

这一部分主要用Erays对以太坊生态系统的代码复杂性和重用性作分析。

### 1. Code Complexity

以下是作者对代码复杂性的实验结果。

![figure2](/images/2018-11-28/figure2.png)

![figure3](/images/2018-11-28/figure3.png)

![figure4](/images/2018-11-28/figure4.png)

需要注意的是，分析复杂度上，作者采用了McCabe的Cyclomatic complexity。Cyclomatic complexity 用线性非依赖的路径来量度CFG的复杂度。在Figure 4 中，complexity 有一个ERC-20 代币造成的峰。

### 2.Code Reuse

在对代码重用的分析上，作者先对指令做了处理。因为指令中的常量操作数能给代码的重用比较带来错误（比如说特定的地址和返回值），所以在重用分析里面，就把这些去掉，只留下指令本身。对每一个block都做上述的处理。将blocks按照函数做整理，每一个函数形成一个block set。最后，将block set做**哈希**。对重用性评估时，只去比较哈希。

哈希这一步大大降低了比较的时间复杂度。

此外，作者还对函数的名称做了调查，并且统计了这些同名函数的实现个数。结果如下。

![table2](/images/2018-11-28/table2.png)

### 3.Reducing Contract Opacity

有了上面这些工作，就可以用来减低闭源合约的不透明度了。方法即为将编译好的bytecode经过上面的处理后，和区块链上经过上面处理得到的数据集作比较，匹配成功则恢复成功。

实验结果如下图所示。

![table3](/images/2018-11-28/table3.png)

注意到有一个合约完全被复原了。

## Reverse Engineering Case Studies

这一部分作者逆向了4个合约作为例子，展示了Erays用途。四个例子为：

- Access Control Policies of High-Value Wallets
- Exchange Accounts
- Arbitrage Bots on Etherdelta
- De-obfuscating Cryptokitties

## Conclusion

因为以太坊有部分智能合约闭源，而其牵扯的交易和金额都相当可观，故作者开发了Erays逆向工具。该工具能将bytecode转换为高级的伪代码。此外，Erays能用来分析代码的复杂性和重用性，并且通过和开源的合约对比来降低闭源合约的不透明度。最后，用4个逆向例子来展示Erays的用途。并且以对这几个例子的分析得出“security by obscurity”的结论。
