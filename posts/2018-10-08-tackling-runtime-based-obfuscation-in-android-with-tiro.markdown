---
layout: post
title: "Tackling runtime-based obfuscation in Android with TIRO"
date: 2018-10-08 15:36:25 +0800
comments: true
categories: 
---

![1534825536001](/images/2018-10-08/1534825536001.png)

出处：USENIX 18'

资料：[Slides](https://www.usenix.org/sites/default/files/conference/protected-files/security18_slides_wong.pdf)、[Paper](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-wong.pdf)

# 1 Abstract & Introduction

混淆技术经常被使用在恶意软件上对抗自动化的程序分析。在Android 平台恶意软件经常使用Java反射、加壳还有字符串加密等方式。作者把常见的混淆技术如字符串加密、动态解密、Java反射调用native方法归类为`language-based`混淆，而完全避开Java完全在Native代码中执行的混淆方式叫做`full-native`混淆。作者在文中提出了一种新型的混淆技术，它会破坏Android运行时完整性的同时使用混淆技术，作者称之为`runtime-based` 混淆。`runtime-based`混淆优于`language-based`混淆和`full-native`代码混淆。 虽然`language-based`的混淆技术必须在调用混淆代码之前立即发生，但`runtime-based`混淆技术可以在同地方发生，并在应用程序看似无关的部分中改变代码执行。一旦程序的完整性被破坏，在运行时就不会遵循正常情况下所期望的代码执行和方法调用流程，`runtime-based`混淆修改了方法调用的解析方式和代码执行方式。

作者提出了一个Android平台去混淆的框架TIRO(Target-Instrument-Run-Observe)。TIRO既可以对抗传统的混淆方式，也可以处理`runtime-based`混淆技术。作者最后测评了来自VT的2000个恶意软件样本，最后有80%左右的样本都使用了runtime-based混淆。

<!--more-->

# 2 Background

###反射

Java提供了使用反射动态实例化和调用方法的功能。 因为反射方法调用的目标仅在运行时才能获取，这样就会提高静态分析的门槛，并且会隐藏被调用的目标函数和数据。

###数值加密

关键数值和字符串加密存放，在调用之前使用解密方法解密

### 动态加载

使用`dalvik.system.DexClassLoader` 和`dalvik.system.DexFile`中的API动态加载dex文件。

### Native方法

使用ndk实现一些功能，在运行时使用JNI接口去调用native lib执行相关的恶意行为和函数调用

###`Full-native`代码混淆

对native lib进行混淆，针对ELF文件的混淆在PC平台有很长的历史

# 3 Runtime-based obfuscation

作者在描述基于运行时的混淆之前，首先介绍了如何在ART运行时中加载和执行代码。 

## 3.1 Loading and Execution in the ART runtime
图1说明了加载和调用代码的三个主要步骤。 

- A显示了如何识别DEX字节码并将其从磁盘加载到runtime。 
- 应用程序实例化类时触发B，如何找到DEX文件中的相应字节码并将其合并到runtime状态。
- C是指如何通过虚方法表（vtable）动态解析虚方法，并将执行指向目标方法代码。

![1536392292718](/images/2018-10-08/1536392292718.png)

##3.2 Obfuscation techniques

基于运行时的混淆通过在上图代码加载和执行过程期间在多个地方处破坏runtime状态来重定向方法调用。 因为基于运行时的混淆通过修改运行时的状态来工作，所以一般需要使用反射获取所需要修改的runtime对象的地址，并通过JNI调用的native lib修改（因为Java内存管理会 防止Java中的代码修改ART运行时对象）。 

作者通过分析已经确定了恶意软件使用的六种不同技术来混淆方法调用的目标。 在图1中，1-3表示可以修改的运行时状态劫持代码加载过程，以便使用非预期（unexpected）数据初始化状态。4-6表示可以破坏运行时状态，以更改方法到调用解析的代码。

###①②DEX file hooking

在加载DEX文件时，dalvik.system.DexFile类在Java代码中用于标识加载的文件。但是大部分实际加载是由runtime中的native代码art::DexFile类执行的。DexFile::mCookie实际是一个指向表示此DEX文件的native代码中art :: DexFile实例的指针。

混淆技术可以使用反射来访问私有mCookie字段并将其重定向到另一个art::DexFile对象，使用包含恶意代码的DEX文件替换原有的
DEX文件。在大多数情况下，恶意DEX文件使用本机代码中的非API方法和类加载，或者在内存中动态生成，进一步隐藏其存在。

同时混淆代码也可以修改art::DexFile本机类中的begin_字段，并将其重定向到另一个DEX文件，而不是修改mCookie字段。

### ③Class data overwriting

混淆代码也可以直接修改映射到内存中的DEX文件的内容，修改class data pointer（class_data_item）可以让混淆代码使用一个恶意的Class对象替换Class对象的定义，修改method data pointer可以让混淆代码修改实现Class对象中某个方法的位置。

### ④ArtMethod hooking

混淆代码可以使用反射获取Class对象的句柄，并确定存储vtable的偏移量。 通过修改此表中的条目，可以映射到被调用的目标ArtMethod对象，然后就可以执行不同的方法。

### ⑤Method entry-point hooking

一旦确定了被调用目标ArtMethod对象，就通过调用其一个入口点来执行该对象，这些入口点仅仅是函数指针。混淆代码通过修改这些入口点的值，来更改调用方法时执行的代码。

###⑥Instruction hooking and overwriting

最后一种方法是在执行DEX或OAT代码指令的时候。这些代码指针是从ArtMethod对象存储和检索的。可以通过修改该指针来实现指令的覆盖或修改，以便在调用该方法的时候执行不同的指令集。

#4 TIRO: A hybrid iterative deobfuscator

TIRO结合了动态和静态两种分析方式去对抗混淆。TIRO的输入是一个APK文件，输出则是一系列混淆信息。这些信息可以作为其他安全分析工具的输入，或者由安全人员进行直接分析。TIRO的运行分为以下四个部分：

###`T`arget

使用静态分析去定位可能发生混淆的地方：

1. language-based:reflection APIs等
2. rruntime-based:native 调用

### `I`nstrument

对如上的定位进行插桩，目的是为了获得去混淆所需要的必要信息

### `R`un

执行混淆代码，并触发app去混淆然后执行代码

###`O`bserve

观察并收集插桩所输出的去混淆信息。 如果发现去混淆显示更多混淆代码则迭代上述步骤，直到执行了所有可能包含混淆的目标位置。

# 5 Implementation

##5.1 AOSP modifications

作者的动态执行环境是基于AOSP定制art/runtime和libcore/libart之后的系统，包括以下三个版本：4.4 (KitKat), 5.1 (Lollipop), and 6.0
(Marshmallow)。因为这三个版本的Android上的DEX文件hook技术存在不同差异（差异是由Android虚拟机引起的）。

##5.2 Extending IntelliDroid

TIRO使用IntelliDroid的静态分析技术定位可能出现混淆的代码，并且使用它的客户端动态地生成输入以能够触发这些位置的代码。为了能够支持ART，作者还对其进行了修改以便运行在Android 4.3-6.0的系统上。

##5.3 Soot modifications

作者使用Soot来对Dalvik bytecode 文件进行读写和修改，为了将去混淆值重新纳入TIRO的静态部分，作者对Soot进行了一些修改。如果在运行时字节码被修改，TIRO会把修改的部分提取出来并在随后的迭代中对它们进行检测。

# 6 Evaluation

作者标记了来自谷歌的34个恶意软件样本，每一个样本都用22种不同的混淆工具混淆过。作者与Google分享了TIRO的检测结果，他们会对作者的结果进行确认。作者首先评估TIRO的准确性，并详细说明TIRO对标记数据集的研究结果。然后为了检测野外恶意软件，作者将TIRO应用于来自VirusTotal的2000个恶意软件样本。

##6.1 General findings



![1536398921883](/images/2018-10-08/1536398921883.png)

作者针对谷歌的恶意软件应用分析如下：

1. 所有的样本都使用了反射，大约53% (18/34)的样本使用了runtime-based混淆，所有的样本都混淆 2-4层
2. 34个样本中的21个包括TIRO的代码欺骗能够规避的代码完整性检查。 大多数样本在加载后会删除解密的代码文件。 
3. 混淆通常用于隐藏对Java中敏感API的调用，这些API用于执行恶意活动。例如SMS的滥用和对敏感数据的访问，包括位置信息和设备
   ID。

##6.3 Evaluation on VirusTotal dataset

作者对2000个VT样本的分析结果如下：

![1536399410757](/images/2018-10-08/1536399410757.png)
