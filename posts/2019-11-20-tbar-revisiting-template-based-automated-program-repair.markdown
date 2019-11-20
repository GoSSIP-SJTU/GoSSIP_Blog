---
layout: post
title: "TBar: Revisiting Template-Based Automated Program Repair"
date: 2019-11-20 02:37:12 -0500
comments: true
categories: 
---
> 作者：Kui Liu, Anil Koyuncu, Dongsun Kim, and Tegawendé F. Bissyandé
>
> 单位：University of Luxembourg
>
> 出处：ISSTA'19
>
> 原文：<https://brucekuiliu.github.io/papers/liu2019tbar.pdf>

<hr/>

## Abstract

在本文中，作者重新评估了已有基于模板的自动化程序修复技术（APR: automated program repair）的性能，以全面了解各类修复模板（fix pattern）的有效性，并强调其它例如故障定位（fault localization）以及贡献代码提取（donor code retrieval）等辅助步骤的重要性。

为了实现这一目标，作者首先调查了已有文献，并收集、总结、标记出了各类经常被使用到的修复模板。基于此调查，文章提出了TBar，其系统的尝试直接使用这些修复模板来修复程序错误。作者在Defects4j测试集上全面评估了TBar，尤其是不同修复模板在生成合理的补丁（plausible patch）以及正确的补丁（correct patch）的有效性上的实际差异。

作者最终发现，在完全准确的故障定位的前提下，TBar正确/合理的修复了74/101处错误。在实际上的标准APR技术评估过程中，TBar在Defects4j测试集中正确的修复了43处错误，取得了在已有文献中前所未有的性能。

<!--more-->

## 1 Introduction

APR技术的一种早期策略便是基于各类修复模板来生成补丁代码。现在这种策略在已有文献中已经十分常见，并且已经在多个APR系统中得到了实现。尽管已有文献记录了基于模板的APR技术的评估结果，但并没有任何广泛评估各类修复模板有效性的研究出现。

文章全面研究了不同修复模板对于自动化程序修复的有效性，并从以下三个方面评估了各类修复模板：

+ 多样性：已有基于模板的APR技术所使用的修复模板的差异是怎样的？
+ 修复性能：不同修复模板的有效性是怎样的？
+ 对于故障定位噪声的敏感度：不同修复模板对于故障定位的误报的敏感度是一样的吗？

总的来说，文章的研究得出了以下发现：

+ 创纪录的性能：TBar创造了一个更高的自动化程序修复性能基准。在完全准确的故障定位的前提下，TBar正确/合理的修复了74/101处错误。在实际的故障定位的前提下，TBar正确/合理的修复了43/81处错误；
+ 修复模板选择：大多数的错误均只能被单一的修复模板正确修复，而其它的修复模板则只能产生合理但不正确的补丁代码。这表明恰当的修复模板使用顺序能够防止产生合理但不正确的补丁代码；
+ 修复原料提取：贡献代码是使用修复模板进行自动化程序修复时的原料，对于基于模板的APR技术来说，选取合适的贡献代码来进行自动化程序修复是极具挑战性的。不合适的贡献代码可能会产生合理但不正确的补丁代码；

+ 故障定位噪声：在使用修复模板进行自动化程序修复时，故障定位的精度对于最后的修复性能具有很大的影响，例如在错误的位置使用修复模板可能会产生合理但不正确的补丁代码。

## 2 Fix Patterns

作者系统的查阅了已有APR文献，并从中筛选出了使用模板的APR技术。文章关注的是修复Java程序错误的基于模板的APR技术，作者人工从这些技术的论文描述以及相关代码实现中搜集得到了所有被明确提及的修复模板。

![20191116131626](/images/2019-11-20/20191116131626.png)

### 2.1 Fix Patterns Inference

已有基于模板的APR技术主要通过以下四种方式来总结得到修复模板：1）人工总结；2）数据挖掘；3）预先定义；4）统计特征。

### 2.2 Fix Patterns Taxonomy

作者人工检查了已有APR文献中出现的所有修复模板，基于代码上下文（例如一条赋值语句）、代码修改操作（例如插入一条进行“instanceof”检查的“if”语句）、以及目标（例如防止程序抛出ClassCastException异常），标记并识别出了15类修复模板。每类修复模板可能会包含一个或者多个特定的子类修复模板。

**FP1. Insert Cast Checker**

```
        + if (exp instanceof T) {
            var = (T) exp; ......
        + }
```

**FP2. Insert Null Pointer Checker**

```
FP2.1: 	+ if (exp != null) {
			...exp...; ......
		+ }
FP2.2: 	+ if (exp == null) return DEFAULT_VALUE;
		  ...exp...;
FP2.3: 	+ if (exp == null) exp = exp1;
		  ...exp...;
FP2.4: 	+ if (exp == null) continue;
		  ...exp...;
FP2.5: 	+ if (exp == null)
		+ 	throw new IllegalArgumentException(...);
		  ...exp...;
```

**FP3. Insert Range Checker**

```
        + if (index < exp.length) {
        	...exp[index]...; ......
        + }
OR
        + if (index < exp.size()) {
        	...exp.get(index)...; ......
        + }
```

**FP4. Insert Missed Statement**

```
FP4.1: 	+ method(exp);
FP4.2: 	+ return DEFAULT_VALUE;
FP4.3: 	+ try {
			statement; ......
		+ } catch (Exception e) { ... }
FP4.4: 	+ if (conditional_exp) {
			statement; ......
		+ }
```

**FP5. Mutate Class Instance Creation**

```
          public Object clone() {
        - 	... new T();
        + 	... (T) super.clone();
          }
```

**FP6. Mutate Conditional Expression**

```
FP6.1: 	- ...condExp1...
		+ ...condExp2...
FP6.2: 	- ...condExp1 Op condExp2...
		+ ...condExp1...
FP6.3: 	- ...condExp1...
		+ ...condExp1 Op condExp2...
```

**FP7. Mutate Data Type**

```
FP7.1: 	- T1 var ...;
		+ T2 var ...;
FP7.2: 	- ...(T1) exp...;
		+ ...(T2) exp...;
```

**FP8. Mutate Integer Division Operation**

```
FP8.1: 	- ...dividend / divisor...
		+ ...dividend / (double or float) divisor...
FP8.2: 	- ...dividend / divisor...
		+ ...(double or float) dividend / divisor...
FP8.3: 	- ...dividend / divisor...
		+ ...(1.0 / divisor) * dividend...
```

**FP9. Mutate Literal Expression**

```
FP9.1: 	- ...literal1...
		+ ...literal2...
FP9.2: 	- ...literal1...
		+ ...exp...
```

**FP10. Mutate Method Invocation Expression**

```
FP10.1: - ...method1(args)...
		+ ...method2(args)...
FP10.2: - ...method1(arg1, arg2, ...)...
		+ ...method1(arg1, arg3, ...)...
FP10.3: - ...method1(arg1, arg2, ...)...
		+ ... method1(arg1, ...)...
FP10.4: - ...method1(arg1, ...)...
		+ ...method1(arg1, arg2, ...)...
```

**FP11. Mutate Operators**

```
FP11.1: - ...exp1 Op1 exp2...
		+ ...exp1 Op2 exp2...
FP11.2: - ...(exp1 Op1 exp2) Op2 exp3...
		+ ...exp1 Op1 (exp2 Op2 exp3)...
FP11.3: - ...exp instanceof T...
		+ ...exp != null...
```

**FP12. Mutate Return Statement**

```
        - return exp1;
        + return exp2;
```

**FP13. Mutate Variable**

```
FP13.1: - ...var1...
		+ ...var2...
FP13.2: - ...var1...
		+ ...exp...
```

**FP14. Move Statement**

```
        - statement;
          ......
        + statement;
```

**FP15. Remove Buggy Statement**

```
FP15.1:   ......
		- statement;
		  ......
FP15.2: - methodDeclaration(Arguments) {
		- 	......; statement;......
		- }
```

### 2.3 Analysis of Collected Patterns

作者从定性和定量两个方面对于搜集得到的修复模板进行了分析。其中，定性分析考虑了以下四个定性维度：

+ 修改操作：修复模板使用了哪种高层操作来修复错误代码？
+ 修改粒度：修改操作直接影响了哪种代码实体？
+ 错误上下文：代码中的哪种特定AST节点被用来匹配修复模板？
+ 修改范围：每类修复模板会影响多少条代码语句？

![20191116144122](/images/2019-11-20/20191116144122.png)

![20191116144325](/images/2019-11-20/20191116144325.png)

## 3 Setup for Repair Experiments

### 3.1 TBar: a Baseline APR System

![20191116160641](/images/2019-11-20/20191116160641.png)

作者期望APR社区将TBar作为APR工具的基准：新的APR系统必须提出能够解决其它附属问题（例如修复精度、搜索空间优化、以及故障定位重排序等）的新技术，以加速自动化程序修复的过程，使其性能高于直接使用常见修复模板的基准APR系统。

**Fault Localization**

TBar利用GZoltar框架来自动执行每个错误程序的测试用例，并使用Ochiai排序指标来计算每条可能是错误代码位置的代码语句的可疑分数。

**Fix Pattern Selection**

TBar是基于每条可疑代码语句的AST上下文信息来进行修复模板的选择的。

**Patch Generation and Validation**

如果经过修复的程序成功通过了所有的测试用例，那么此补丁代码就被认为是一个合理的补丁（plausible patch）。如果一个合理的补丁与实际的补丁在语义上是一致的，那么此补丁代码就被认为是一个正确的补丁（correct patch）。

当一个AST节点与多个修复模板的上下文相匹配时，TBar将按照Update > Insert > Delete > Move的高层操作优先级来使用这些修复模板。当一类特定的修复模板与多处贡献代码相匹配时，TBar将按照贡献代码节点与错误代码节点在错误程序的AST中的距离远近来选择这些贡献代码，距离近的贡献代码的优先级更高。

### 3.2 Assessment Benchmark

在实验评估中，作者选择了Defects4J测试集来验证测试TBar。

![20191116165405](/images/2019-11-20/20191116165405.png)

## 4 Assessment

作者进行了以下两组实验来评估TBar的自动化程序修复性能：

+ Experiment #1：测试TBar所实现的各类修复模板的有效性。为了避免故障定位的误报引入偏差，作者直接为TBar提供了完全准确的故障定位信息；
+ Experiment #2：在实际的程序修复情境下评估TBar的性能，尤其是各类修复模板产生错误的补丁代码的数量与趋势。

### 4.1 Repair Suitability of Fix Patterns

![20191116183335](/images/2019-11-20/20191116183335.png)

![20191116184444](/images/2019-11-20/20191116184444.png)

搜集得到的修复模板能够被TBar用来修复Defects4J测试集中的74处错误。然而，测试集中还有很大一部分的错误无法被TBar修复，这在很大程度上是由于：1）修复模板集合的局限性；2）查找并提取相关修复原料以根据修复模板生成补丁代码的策略的局限性。

![20191116184513](/images/2019-11-20/20191116184513.png)

有些程序错误能够被不同的修复模板合理的修复。然而，在大多数情况下，只有一类修复模板能够生成正确的补丁代码。这表明修复模板使用顺序的选择还需要进一步的深入研究。

![20191116191226](/images/2019-11-20/20191116191226.png)

除了修复模板以外，从贡献代码中提取得到的修复原料也同样需要被正确的选择，从而避免产生合理但不正确的补丁代码。

![20191116185555](/images/2019-11-20/20191116185555.png)

修复模板生成的正确补丁代码之间存在明显的差异，这些差异取决于修复模板实现的修改操作、修改粒度、以及修改范围。

### 4.2 Repair Performance Comparison: TBar vs State-of-the-art APR tools

![20191116190135](/images/2019-11-20/20191116190135.png)

TBar的自动化程序修复性能优于所有在Defects4J测试集上评估的已有APR技术，其能够正确修复测试集中的43处错误，而排在第二的SimFix则只能修复测试集中的34处错误。

![20191116190203](/images/2019-11-20/20191116190203.png)

![20191116190228](/images/2019-11-20/20191116190228.png)

![20191116190253](/images/2019-11-20/20191116190253.png)

故障定位噪声对于TBar的性能具有很大的影响。不同修复模板对于故障定位的误报的敏感度各不相同。


