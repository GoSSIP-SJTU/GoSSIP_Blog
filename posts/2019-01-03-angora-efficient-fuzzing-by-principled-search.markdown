---
layout: post
title: "Angora: Efficient Fuzzing by Principled Search"
date: 2019-01-03 16:36:29 +0800
comments: true
categories: 
---

作者：Peng Chen, Hao Chen  

单位：ShanghaiTech University, University of California, Davis  

原文：[https://spinpx.com/papers/Chen2018Angora.pdf](https://spinpx.com/papers/Chen2018Angora.pdf)   

出处：IEEE S&P'18

## 贡献  
作者实现了一个叫做Angora的fuzzer，其具有以下特点： 

- 用上下文敏感的方式进行分支覆盖 
- 利用字节级别的taint tracking分析数据  
- 使用梯度下降来求解条件语句  
- 能够分析数据类型是否带符号和长度  
- 能够探测输入的长度  


## 思路  
### 上下文敏感的分支计数  
AFL所使用的上下文不敏感的分支处理，会使得其不能识别相同分支潜在的不同内部状态。相较于AFL只将分支的始块和终块作为分支特征，作者在此基础上又加入了上下文特征。  
利用对栈进行hash的方法，来获取上下文。为了避免产生过多的独立分支，作者使用的hash函数会异或调用栈。 
如Figure 3，f中的x参数是输入的一部分，而trigger所控制的分支，受“是否第一次调用f”这个上下文环境影响。该分支是一种内部状态。   

<!--more-->

![figure3](/images/2019-01-03/pics/figure3.png)  
### 字节级的污点追踪  
在污点追踪中，作者使用某一字节在输入中的偏移作为taint lable。为了减小taint lable的大小，该方案维护一个表，这个表的索引是taint lable，而它的值是一个二叉树的结点，通过回溯二叉树到根节点的路径，可以得到一个由0和1组成的bitmap，这个bitmap所代表的信息是相对于这个taint label对应的偏移而后第某位个字节的污染情况。通过这种污点分析，可以得到输入的使用情况，继而判断各个字节作为变量的长度，以便进一步的进行数据突变等操作。
### 基于梯度下降的搜索算法  
在已经分析出变量的前提下，要想获得更多分支，就需要构造变量使得输入满足进入新分支的条件。如果使用符号执行的方法，成本太高。作者选择使用梯度下降的求解的方法来获得满足条件语句的值。  
如Table 2，作者将条件判断语句转换成误差函数，尔后利用梯度下降的方法求解误差函数，进一步的可实现对条件语句的满足。  

![table2](/images/2019-01-03/pics/table2.png)
### 变量大小与类型判断  
变量在污点分析时已经判断过大小了，而类型（是否带符号）可以通过语义判断。当明确了变量的大小和类型后，基于条件语句构造的误差函数中变量的定义会更准确。
### 输入长度探测  
输入太短没有效果，太大会爆内存。因此需要得出合适的输入长度。作者认为只有当增加长度能得到新分支时，才增加长度，也就是使得读取长度尽可能满足程序的需要。具体做法是，对于类似read的函数，根据它的读入长度是否会影响条件语句的结果来决定扩大长度与否。  
## 实验  
在和其他fuzzer的比较中，fuzzer运行使用CPU核心数这一变量被统一限定为单核（注：作者提到Angora支持多核，但是实验中未作改变核心数的纵向比较）。每种实验做5次，取平均结果。  
### 通过LAVA语料与其他fuzzer比较  
LAVA技术可以用来在源码中生成现实bug，LAVA-M语料集在每个程序里添加了许多bug。LAVA-M包含了四种GNU coretutils程序：uniq, base64, md5sum, who。作者使用这些语料来做和其他程序做对比测试。  
结果如Table 1，Angora对于发现bug的能力明显优于现有fuzzer，甚至发现了一些在LAVA作者预期内，但是没做处理的BUG。  

![table1](/images/2019-01-03/pics/table1.png)  
对于Angora明显强于较好的其他fuzzer（VUzzer，Steelix）的原因，作者认为有两点  
- Angora追踪了输入字节的偏移  
- Angora对条件表达式做了有力的计算  

这两个机制的结合，比竞品使用“魔数”或者其它预测特殊输入的方法来的更有效。  
​    
通过Figure 4我们可以看出Angora的时间效率，本图测试样本为who，在前5分钟就可以发现近1000个bug，虽然之后发现bug的速率下降，但是45分钟的运行已经能让他发现超过1500个bug。值得一提的是，回顾Table 1，Angora在who上发现的bug数远超其他fuzzer，其中包括98个LAVA作者预期内但是没有做处理的bug。  

![figure4](/images/2019-01-03/pics/figure4.png)  
### 通过未修改的现实软件测试Angora  
作者选用AFL和Angora做对比，由于测试的是现实软件，bug较少，所以作者除了测试发现新bug，还对运行覆盖情况做了测试。其中对新bug发现能力的检测，以触发unique crash的数量为标准。      
分别经过5小时的运行，Angora发现的unique crash数多倍于AFL。同时，通过表5我们可以看出，Angora在代码覆盖率上对AFL有所改进，在jhead做样本时的实验中Angora在行覆盖和分支覆盖上，分别有127.4%和144.0%的增长。  

![table5](/images/2019-01-03/pics/table5.png)  
### 上下文敏感的分支计数  
- 效果  
在前文中提到，作者认为，如果同一分支能以来源于不同的调用者区分开来（改分支计数部分），那么fuzzer可以找到更多bug。为了验证这一假设，作者分别使用是否带有上下文敏感机制的Angora，来测试file程序。  
结果是带有上下文敏感的Angora在代码覆盖上不明显强于非上下文敏感，但是带有上下文敏感的Angora可以发现6个unique crash，而不带有的不能。  
- hash碰撞  
Angora和AFL都使用了hash表来存储分支，增加上下文敏感机制使得unique分支的数量至多是不带有上下文敏感机制时的8倍左右。因此，作者使用了16倍于AFL的hash表大小的hash表。尽管Angora的hash表大小与cache大小不适配，他的查找机制使得这种不适配的性能影响得到减弱。  
### 基于梯度下降的搜索算法  
对限制条件进行梯度下降求解比随机突变和魔数突变的方法（改5.1）能求解更多的条件表达式，如Table 8所示。  

![table8](/images/2019-01-03/pics/table8.png)  
没使用符号执行，使得在fuzz大程序上更有效率  

### 输入长度探测
Angora对比AFL的随机长度机制，以更少的增长次数等到更多的有用路径，同时，更短的平均执行长度，使得Angora运行得更快。  

![table9](/images/2019-01-03/pics/table9.png)  
### 执行速度  
相较于AFL，如果没有taint机制，Angora与AFL的插桩机制拥有相同的运行速度。从Table 10可以看出，在taint机制的影响下，Angora的运行效率比AFL稍低，相同时间处理的输入较少。不过结合之前的实验，在fuzzer的执行结果方面，Angora效率更高。  

![table10](/images/2019-01-03/pics/table10.png)  
