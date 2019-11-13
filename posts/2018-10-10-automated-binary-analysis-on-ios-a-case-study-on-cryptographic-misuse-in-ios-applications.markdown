---
layout: post
title: "Automated Binary Analysis on iOS – A Case Study on Cryptographic Misuse in iOS Applications"
date: 2018-10-10 12:40:09 +0800
comments: true
categories: 
---



作者: Johannes Feichtner, David Missmann, Raphael Spreitzer

出处: WiSec'18

单位: Graz University of Technology

这篇论文介绍了一种在iOS应用程序中发现密码学误用的方法。他们通过将64位ARM二进制文件反编译为LLVM中间表示（IR），接着应用静态程序切片来确定相关代码段中的数据流，然后将提取的执行路径与一组预定义的安全规则相比较，最后确认iOS应用程序中存在问题的语句。

<!--more-->

### 相关背景

#### iOS Applications
Mach-O的文件格式

*   Header
*   Load Commands
*   Data

方法调用
*   __objc_classlist
*   __objc_data
*   __objc_const

Dynamic Loader Info：包括不在文件中的指向类的指针。

在执行期间，应用程序可以使用间接符号调用外部定义的库函数。

#### Program Slicing
静态切片可用于确定程序的代码语句，这些语句可能影响指定执行点的值。 生成的程序片涵盖所有可能的执行路径。 

Weiser提出了一种程序内方法，该算法包括两个步骤：

*   Follow data dependencies:
*   Follow control dependencies

要在多个函数上创建切片，可以将该方法扩展到过程间切片。

#### Pointer Analysis
指针分析可用于通过关于指针状态的准确信息来帮助切片过程。

Andersen提出，指针分析被描述为集合约束问题，为给定程序创建约束系统。通过求解系统，可以确定变量在程序执行期间可能指向的位置。存在四种不同类型的约束。
![-w330](/images/2018-10-10/media/15310425477164/15311100077818.jpg)


#### Cryptography on iOS
iOS上有密码函数的链接库有CommonCrypto和Security，包括以下函数。
```
CCCryptorCreate
CCCrypt
CCKeyDerivationPBKDF
```

### 系统设计
![-w346](/images/2018-10-10/media/15310425477164/15311101766076.jpg)

1. Disassemble
2. Decompile
3. Pointer Analysis
4. Static Slicing
5. Parameter Backtracking

#### 反编译到LLVM IR
此步骤将64位ARMv8二进制文件转换为LLVM IR代码。

LLVM编译器前端将源代码作为输入。在对代码进行解析和分析之后，生成LLVM IR代码。然后，编译器后端负责对其进行优化，组装机器代码并链接生成的对象。

作者构建了逆向工程框架Dagger。与类似的方法相比，它扩展了LLVM并依赖于指令语义，根据寄存器和指令的目标描述将机器指令转换为LLVM IR代码。

#### 恢复丢失的信息
*   Intraprocedural Control Flow
*   Function Parameters and Return Values
*   External Symbols

#### 实现
*   Registers
*   Non-Volatile Registers
*   Tail Calls

### 指针分析
#### Iterative Constraint Generation
*   二进制偏移量可以通过inttoptr转化为静态地址访问。
*   堆地址是通过内存分配函数创建的，它可以立即用于指针分析。
*   通常通过向栈指针添加偏移来访问堆栈存储器。

#### Objective-C特点
objc_msgSend是Objective-C运行时库中的一个函数，负责在运行时决定调用哪种方法。它需要两个参数来指定类或对象，以及要调用的方法的名称。必须通过为调用objc_msgSend的方法添加边来恢复正确的调用图。

### 静态切片
静态切片的目的是计算影响切片标准的所有代码段。 作者采用Weiser的算法，通过考虑ARMv8的特性来处理LLVM IR代码。

由于参数可以使用ARMv8上的栈或寄存器传递，因此Weiser的方法不能直接适用于处理LLVM IR代码。 针对此问题的解决方案是扩展所有相关变量ROUT(i)的集合，其中包含在函数调用之前读取或修改寄存器的加载和存储操作的信息。

![-w416](/images/2018-10-10/media/15310425477164/15311126675361.jpg)

#### Restoring Missing Type Information
在反编译步骤中恢复函数参数和返回值之后，实例变量和协议方法仍然缺少类型信息。

通过从二进制文件中的偏移地址来访问实例变量。这使我们能够找到引用特定变量的所有指令。但是二进制文件没有针对Objective-C或Swift中协议声明的方法参数的精确类型信息。虽然为每个参数创建抽象位置不允许进行精确的指针分析，但仍然可以识别使用这些对象进行的调用。

#### Parameter Backtracking
*   Finding Predecessors
*   Extracting Execution Paths
*   Avoiding Cycles

### 检测密码学误用
*   Do not use ECB mode for encryption
*   Do not use a non-random IV for CBC encryption
*   Do not use constant encryption keys.
*   Do not use constant salts for PBE
*   Do not use fewer than 1,000 iterations for PBE
*   Do not use static seeds to seed SecureRandom

### 评估
#### Manual Analysis
此步骤的目标主要是测试所有组件是否相互兼容。将框架应用于开源应用程序，并根据源代码检查获得的结果。

对于此分析，从GitHub下载了15个使用CommonCrypto进行加密的开源应用程序。其中8个是密码管理器，其余应用程序有电子邮件，数据容器，云存储等。

#### Automated Analysis.
此步骤的目的是调查iOS应用程序开发人员是否知道如何正确应用加密API。

作者从官方iOS App Store下载了634个免费应用程序。在获取它们之后，在越狱的iPhone上使用工具Clutch来解密应用程序。事实证明，495（78％）的应用程序包括对加密API的调用。但是，剩余的139个应用程序也包括密码管理器和使用密码术的应用程序，这可能是由于这些应用使用了自己实现加密例程的第三方库造成的。

#### Results
总的来说，评估了495个闭源和15个开源应用程序，其中包括对CommonCrypto的调用。 总共417 + 15 = 432个应用程序，分析工作流程成功完成。 
![-w345](/images/2018-10-10/media/15310425477164/15311116991143.jpg)

表2是78个闭源应用程序的分析失败的原因。 7个应用程序仅包含ARMv7平台的二进制文件。 其次，由于缺少指令语义，9％的应用程序无法反编译。

### 总结
在这项工作中，作者开发了一个多步骤方法来促进iOS平台上的密码学误用分析。将64位ARMv8二进制文件转换为LLVM IR代码。通过从二进制文件重建丢失的信息，精确地建立控制和数据流图，以用于程序切片和参数回溯。

基于此框架，作者分析了大量的iOS应用程序，来检测安全关键密码函数的误用。作者发现417个被调查的应用程序中有343个违反了至少一个安全规则。
