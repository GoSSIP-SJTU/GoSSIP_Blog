---
layout: post
title: "SandScout- Automatic Detection of Flaws in iOS Sandbox Profiles"
date: 2018-04-16 15:33:09 +0800
comments: true
categories: research_paper
---

作者：Luke Deshotels, Ra ̆zvan Deaconescu, Mihai Chiroiu, Lucas Davi, William Enck & Ahmad-Reza Sadeghi  
原文：[CCS'16](https://dl.acm.org/citation.cfm?id=2978336)  
单位：North Carolina State University, University POLITEHNICA of Bucharest & Technische Universität  

## Contributions
* 开发了一款工具以自动化分析iOS的沙箱规则并生成人类可理解的SBPL规则。首先，工具会自动化解压、反编译iOS中的沙盒描述文件到SandBox Profile Language（SBPL）
* 使用Prolog对SBPL规则进行了建模。将SBPL转换为Prolog的facts之后，借助Prolog对沙箱配置进行建模；最后通过他们设计的Prolog查询来对这些沙箱规则进行安全性测试并且用一个iOS APP对检测到的安全问题进行验证。
* 对iOS的沙箱描述进行了系统化测试。通过他们的工具发现了iOS 9.0.2下的七个漏洞。列举如下

```
1. 绕过iOS隐私设置获取联系人信息
2. 获取用户的位置搜索历史信息
3. 通过获取系统文件的meta信息推测敏感数据
4. 获取用户的姓名和媒体库
5. 破坏磁盘空间且不能通过卸载这个恶意app来恢复
6. 禁止访问系统资源，如地址簿
7. 不使用iOS的进程通信机制完成进程间通信
```
<!--more-->
## 背景
#### iOS的四项安全机制
* 应用审查：开发者将APP提交到APP Store上架时，APPLE会先对提交的APP进行审查，审查规则不透明，但普遍认为是静态分析和动态分析同时参与审查。
* 代码签名：开发者提交APP之前自行对提交的APP进行开发者签名，APPLE通过该APP的审查后，对APP文件再次进行签名。当代码在iOS设备上运行时，会首先确认该签名是否有效（一般只承认APPLE的签名，但如果信任了某开发者证书，也可以执行该开发者编译的未发布的代码）。
* 内存保护：DEP、ASLR等传统内存保护机制，同时还对可执行内存进行了严格的写保护。
* 沙箱：iOS沙箱对所有应用设置了访问控制规则，限制了恶意代码的破坏能力。大多数系统应用和daemon都有自己的沙箱描述，第三方APP及少数系统应用使用通用沙箱描述（Container）。

沙箱在进行权限控制时，规则来源主要有两项：entitlements和sandbox extensions。

> entitlements：由开发者在开发过程中定义的静态规则，以字典形式存储。在签名时，也会对entitlements进行签名，这就意味着一旦程序发布后，entitlements就无法再被修改了（除非攻击者有办法对APPLE的证书进行攻击）。

> sandbox extensions：部分系统daemon有权限在程序运行时对程序的沙箱规则进行管理，如tccd，用于帮助用户对隐私设置进行设定，可以修改程序对部分系统资源的访问控制规则。

#### 沙箱描述语言（SBPL）
沙箱描述文件是使用SBPL编写并编译为二进制得到的。

每个SBPL编写的描述文件都包括一个版本号，一个默认决策以及0或者更多条规则。默认决策对应了当没有规则匹配应用的当前行为时，访问控制系统应该做出的决定。Container的描述文件以及大部分沙箱描述文件都使用白名单模式，也就是默认决策为deny。

每个沙箱规则包括一个决策（allow或者deny），一个操作（如file-read-data或者file-write-data），0或者多个filter或者metafilter。

> Filter:由一个filter-type和0或者更多个filter-values组成。filter-type标志了这个filter要匹配内容的类型（比如subpath,literal或者regex），filter-values则描述了对应filter-type的具体值。

> Metafilter:相当于filter的逻辑操作，共有三种类型：require-all（要求所有filter的条件都满足，类似于AND），require-any（要求某一个filter的条件被满足，类似于OR）和require-not（要求所有filter都不被满足，类似于NOT），metafilter可以，而且通常都是，嵌套的。

例子：

```
( allow file-read*
	( require-all
		( subpath "/Media/Safari" )
		( require-not
			( literal "/Media/Safari/secret.txt" )
		)
	( require-entitlement
		"private.signing-identifier"
		( require-any
			(entitlement-value "mobilesafari" ) 			(entitlement-value "safarifetcherd" )
) ) ) )
```

这条规则允许沙箱中的进程读除了`/Media/Safari/secret.txt`以外的所有文件，只要该程序的`entitlement`中`private.signing-identifier`项的值为`mobilesafari`或者`safarifetcherd`。

当然，沙箱规则也可以对`sandbox extensions`中的内容进行规则匹配。

## SandScout
![Overview](/images/2018-04-16/overview.png)

#### Sandbox Decompilation
这项工作的基础是[Blazakis](http)和[Esser](http)的工作，他们将沙箱规则转换成了图的形式来表达，图包含两个终止节点，分别是`Deny`和`Allow`，其他的非终止节点都表示了一个filter，图中的边表示了metafilter（require-all/any/not）是否匹配（实现表示匹配，虚线表示不匹配），整张图表示了一个沙箱操作的规则。

![convering_table](/images/2018-04-16/convering_table.png)

将图转换为SBPL时，用表来进行对应转换，从直接连接`deny`和`allow`的点开始，向上进行转换，直到除了`allow`和`deny`以外，整张图只有一个点为止。



![graph2sbpl](/images/2018-04-16/graph2sbpl.png)

#### Policy Modeling
先将转换完成的SBPL表达式再转换为disjunctive normal form（DNF），因为这个形式本身很容易转换为Prolog。然后他们直接给出了转换结果的实例。

```
allow(file-readSTAR, [literal("/myFile"),extension("A")]).
allow(file-readSTAR, [literal("/myFile"),not(extension("B"))]).
```

这里的转换比较难，有三个原因：
* 每个filter的filter value都可能是一个集合
* metafilter可以嵌套，而且部分filter也会使用metafilter
* 最后必须以DNF的形式输出Prolog的facts

为了解决这几个难点，他们用ply（一个Lex和Yacc的python库）开发了一个从由SBPL直接生成Prolog的上下文无关的编译器。

#### Policy Analysis
这个部分核心是使用Prolog查询（query）进行分析。他们定义了三个查询作为例子来进行测试。

1. 不允许所有APP对系统目录进行写操作
2. 为了保证用户隐私，所有指向`/private/var/mobile`的读操作都需要请求权限
3. 所有请求系统文件读写权限的规则都需要请求权限

PS：他们的查询实际上是去匹配沙箱描述文件转换得到的Prolog是否满足这些条件

#### Attack Verification
他们开发了一个APP来对通过查询找到的漏洞进行测试，同时用APP来测试也可以更快排除前面规则找到的无关漏洞，比如即使部分系统文件能被读写，但是这个文件本身不包含敏感信息，那么这个漏洞就被认为是不重要的。

## 结果
#### Prolog Query Results
测试平台：iOS 9.0.2 on iPhone 5S（jailbroken）
这里的测试对象是Container，但是文中没有具体提到。

SandScout最后产生了1520个Prolog facts，查询结果如下。

![query_results](/images/2018-04-16/query_results.png)

#### Verified Attacks
#####1 绕过隐私设置

```
allow(file-writeSTAR,
	[subpath("/Library/AddressBook/"),
		extension("AddressBook")]).
```
上述Prolog fact被查询1匹配到，因为AddressBook是一个系统路径。允许对这里进行写操作时，攻击者可以在为其创建硬链接，这样即使后来用户取消了对该APP访问地址簿的授权，APP仍然可以通过建立的地址簿来进行写操作。

APPLE的修复方案是，再加入一个file-link的权限设置，来禁止随意添加链接。在iOS 9.1中，他们通过SandBlaster发现了下述语句。

```
(allow file-link
	(require-not
		(subpath
			"/Library/AddressBook")))
```

##### 2 隐私泄露

Container的沙箱描述允许第三方APP读取少量包含用户数据的系统文件，他们认为这些文件中的一部分可以导致信息泄露。具体例子如下：

* iTunes Privacy Leaks：`/private/var/mobile/Medi- a/iTunes_Control/iTunes`可以被任意第三方APP读取，然而这个文件夹下，至少有三个包含隐私数据的文件。a.包含iTunes购买记录的title和元数据的数据库；b.包含用户名和存有当前设备备份的计算机名的文件；c.包含通过iTunes安装的APP的列表文件。
* Maps Privacy Leaks：`/private/var/mobile/Library/- Caches/GeoServices`可以被任意第三方APP地区。这个文件夹下包含了用户通过Apple Maps搜索过的地点名。
* Metadata Privacy Leaks：Container的沙箱描述会创建所有第三方APP可访问的目录和符号链接的元数据，而且这些数据是可以被第三方APP获取的，第三方APP可以通过最后修改时间、大小等数据来推测用户的部分行为。
* 未授权的进程间通信：文件`/private/var/mobile/Media/com.apple.itunes.lock_sync`和`/private/var/mobile/Library/Keyboard/LocalDictionary`可以被所有第三方APP读写，目录`/private/var/mobile/Library/Caches/com.apple.keyboards/`可以被所有第三方APP读写，目录`/private/var/mobile/Library/DeviceRegistry/`可以被所有第三方APP创建以数字和字母组合命名的文件夹。这些都可以被APP用来滥用为APP间通信的通道。

#####3 系统级破坏
空间占用：第三方APP可以通过向系统目录写入大量无用数据来填满系统空间，而且这些空间很难被释放出来。

锁死系统文件：第三方APP可以通过删除系统文件（如果有写权限）然后创建一个同名文件夹来避免系统恢复该文件。

## Limitations
Sandbox Decompilation：SandBlaster不能保证他反编译出来的语义和原来的语义是等同的，这可能导致分析结果出错或者漏掉部分漏洞；另外他需要iOS的固件key被泄露，否则无法解密iOS固件代码，也无法提取沙箱描述文件。

Policy Modeling：他们从SBPL转换到Prolog的模型由四点假设，一是假设SBPL是没有错（written correctly）的，但是后来他们发现对于SandScout来说这个假设其实没有必要；二是假设版本信息会出现在第一行，并且默认决策就在第二行；三是假设他们碰到的filters和metafilters就是需要处理的所有了，一旦有新的(meta)filter加入进来，他们的工具就必须更新了；四是假设require-not的metafilters不会在包含其他metafilter，这个由SandBlaster保证（逻辑运算），以后可以去掉这项假设。

Policy Analysis：他们写的Prolog queies只针对文件访问，对其他的操作的分析留到以后继续做。而且即便是只针对文件访问，他们的工作也可能是不全面的。

Attack Verification：他们用越狱设备做的测试，这样比较方便判断。但是越狱本身可能对测试是有影响的，也就是他们没有测试在未越狱设备上这些攻击是否有效。

































