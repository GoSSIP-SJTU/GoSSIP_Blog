---
layout: post
title: "Retrofitting Fine Grain Isolation in the Firefox Renderer"
date: 2020-06-23 20:58:47 +0800
comments: true
categories: 
---

> 作者：Shravan Narayan, Craig Disselkoen, Tal Garfinkel, Nathan Froyd. Eric Rahm, Sorin Lerner, Hovav Shacham, Deian Stefan
> 
> 单位：UC San Diego, Stanford, Mozilla, UT Austin
> 
> 出处：USENIX Security 2020
> 
> 资料：[Paper](https://www.usenix.org/system/files/sec20fall_narayan_prepub.pdf), [代码](https://github.com/shravanrn/LibrarySandboxing)

## 1. Abstract

Firefox以及一些其他的主流浏览器都用了大量的第三方库用于渲染音频、视频、图片以及其他的一些内容，这些库至今已经被发现过大量漏洞。作者针对这个问题设计了RLBox将Firefox进行改造隔离 (Retrofitting Isolation) ，用轻量级的沙盒对这些库进行保护，能够有效降低这些库被攻击带来的影响，并且可以最小化对Firefox进行保护过程中的人工参与度。

RLBox通过静态信息流以及动态检查 (Dynamic checks) 的方式实现，支持SFI (Software-based-fault isolation) 和进程隔离 (Multi-core process isolation) ，性能开销不高，对网页延时影响也很小 (page latency) 。作者将RLBox用于一些性能很敏感的图片解码库 (libjpeg和libpng) ，视频解码库 (libtheora和libvpx) ，libvorbis音频解码库以及zlib解压库进行实验。目前RLBox已经整合进了生产环境的Firefox，将libGraphite字体处理库进行沙盒化保护。

<!-- more -->

## 2. Introduction & Background

现代主流浏览器都采用粗粒度的方式进行权限隔离，限制漏洞的影响。例如每一个渲染器 (Renderer) 都运行在一个沙盒化的进程里，这个进程即使被攻破了，攻击者也只能做有限的事情，比如不能访问系统资源（装恶意软件）。如下图所示（[图片来源](https://wiki.mozilla.org/File:550px-Sandboxing_basic_architecture.png)）是Mozilla Firefox的进程模型。

![Retrofitting%20Fine%20Grain%20Isolation%20in%20the%20Firefox%20R%20296d1bf3188040489d3989aa2afa2d6c/Untitled.png](/images/2020-06-23/Untitled.png)

但是这还是不够的，Renderer如果被攻击者攻击了，攻击者可能已经可以访问受害者已经认证的Dropbox以及Google Drive这样的云存储服务。在这方面即使浏览器厂商已经花了大量精力寻找Renderer里的漏洞，仍有很多漏洞存在其中。例如Pwn2Own 2018中被用来攻击Firefox的libvorbis越界写漏洞，Chrome和Firefox共同使用的libvpx视频解码库里的整数溢出漏洞，Skia图形库中存在的最近才被发现的四个可以远程代码执行的漏洞。

考虑一个用户在浏览器里用Gmail读取邮件的场景。攻击者Trudy发送了一个含有恶意网站的链接，Alice点击进入后，他的浏览器将会访问Trudy准备的一个嵌有.ogg音频或者.webm视频的网站，然后Trudy将攻破Alice使用的浏览器里的Renderer。Trudy这时就可以控制Alice的Gmail账户，以Alice的身份读取发送邮件，例如可以重置各种网站的密码等，甚至可以跨站点攻击，访问其他Alice已经登录的网站。不过Chrome最新版本已经支持了站点隔离 (Site Isolation) ，将不同的网站进行隔离，例如*.google.com和*.amazon.com已防止这样的跨站点的攻击。

RLBox设计中需要考虑的三个问题：1) 最小化人工参与; 2) 安全性; 3) 效率。

## 3. Fine grain sandboxing

Renderer使用的第三方库很需要也很适合进行细粒度隔离，基于三个原因：

1. 例如图片和视频这样的媒体内容是攻击者很常用的攻击向量。
2. 这些库本身权限都有限，被攻破后最坏情况下只能够控制图片的显示。举反例来说，像Javascript引擎这样的高权限组件进行沙盒化就没什么意义，攻击者如果已经攻破了它，就可以执行任意JS代码，从而控制整个Web应用。
3. 现有的Library-Renderer接口已经天然的将代码进行了分区。

在RLBox之前，Firefox已经使用了粗粒度方法进行库的沙盒化，将所有库都放入一个沙盒化的进程，这个方法在性能上很好，但是安全性上不够，因为整体的安全性取决于安全性最差的库，一旦被攻破，攻击者可以进行跨域攻击。

在本文中，作者按照<renderer, library, content-origin, content-type>的粒度来分配沙盒。

- 每个Renderer都单独一个沙盒可以防止攻击者进行跨Tab和跨浏览器Profile的攻击。
- 每个库单独一个沙盒，可以保证某一个库中存在漏洞不会影响到其他的库。
- 每个Content-origin都单独一个沙盒，可以防止跨域攻击。比如在攻破了sites.google.com的沙盒后，不会影响到pay.google.com。
- 每个Content-type都单独一个沙盒可以解决问题：passive content influencing active content。

**威胁模型**：假设一个web攻击者可以给用户浏览器发送恶意的内容，从而导致了RLBox沙盒内的任意代码执行。保证攻击者只能影响来自相同域的相同类型的内容。测信道在讨论范围之外。

## 4. Pitfalls of retrofitting protection

Firefox Renderer本身是默认三方库是安全的，RLBox则将第三方库认为是不可信的，因此需要修改Renderer-Library接口（过滤不可信的输入）。

作者以Renderer的JPEG解码器与libjpeg的接口为例展示了几个容易出现的问题，如下图所示。

![Retrofitting%20Fine%20Grain%20Isolation%20in%20the%20Firefox%20R%20296d1bf3188040489d3989aa2afa2d6c/Untitled%201.png](/images/2020-06-23/Untitled%201.png)

### **Insecure data flow**

**Failing to sanitize data：**25行处的num_bytes，36行的src，39行的jd和src，如果没有进行检查，都有可能导致任意地址读写。

**Missing pointer swizzles：**44行的指针解引用可能导致任意地址写。

**Leaking pointers：**第4行会泄漏nsJPEGDecoder的地址。

**Double fetch bugs：**第16行和第18行。

### **Insecure control flow**

**Corrupted callback state：**37行

**Unexpected callback invocation：**58行

**Callback state exchange attacks：**42行

## 5. RLBox

目标是修改Firefox JPEG解码器，解决上述提到的问题。人工去完成这个任务是很容易出问题的。RLBox解决了如下问题：

1. 自动化的安全检查：加安全检查是一个劳动密集型的任务，很容易出问题，但是上述提到的一些问题很大都可以通过静态的检查来处理，比如编译时的报错来提示代码需要手动修改以变得更加安全。
2. 不需要对库进行修改：避免修改库，因为第三方库很多，开发者也不想去了解这些库的内部实现。
3. 简化迁移：Firefox用了很多的第三方库，作者希望能减少每个库使用时应用RLBox所需要的花的精力。

RLBox通过tainted类型来保护数据流，用了一个简单的IFC discipline，保证沙盒数据会在被不安全使用之前就被验证，并且防止指针泄漏到沙盒。

RLBox通过tainted类型和所设计的API结合来保护控制流。比如攻击者不能跳转到Tainted类型的值，API则是用来限制Sandbox本身的接口的使用。例如要调用库函数必须通过sandbox_invoke，要从库里回调Renderer的函数，需要通过sandbox_callback实现。

Automate secuity checks：保证来自sandbox的指针都指向的是sandbox的内存。

Minimize renderer changes：tainted类型的数据只有在需要的时候进行验证，RLBox允许一些对taitned类型数据的算术操作或者赋值操作，或者对tainted指针的赋值操作。

Efficiently share data structures：静态检查以保证共享内存只能在sandbox内存里，并且只能通过tainted类型的变量进行操作。

Assist with code migration：编译时类型和接口错误可以指导开发者将库迁移到sandbox里，每一个编译时错误都指向喜爱一个需要修改的代码，比如数据需要在被验证后才能使用，或者需要使用RLBox的API来进行控制流转移。

Bridge machine models：Sandbox机制可以用不通的模型实现，例如Native Client (NaCl)，WebAssembly。

### Data flow safety

所有从sandbox到renderer的数据都是tainted，如下所示。

![Retrofitting%20Fine%20Grain%20Isolation%20in%20the%20Firefox%20R%20296d1bf3188040489d3989aa2afa2d6c/Untitled%202.png](/images/2020-06-23/Untitled%202.png)

从Renderer到Sandbox的数据都是tainted，像指向Renderer内存的untainted指针是不允许传进Sandbox的。

但是这会有一个问题，如何共享内存，作者用如下例子进行解释，即通过sandbox_malloc来分配共享内存。

![Retrofitting%20Fine%20Grain%20Isolation%20in%20the%20Firefox%20R%20296d1bf3188040489d3989aa2afa2d6c/Untitled%203.png](/images/2020-06-23/Untitled%203.png)

### Data validation

所有来自sandbox的数据都被认为是tainted，通过tainted<T>将数据定义为tainted，一旦数据被赋予taint属性后，只能通过显示的validation才能移除。

当Renderer需要使用库返回的数据时，需要进行unwrap，即将tainted<T>转为普通的T类型。RLBox提供了verify接口，如下图所示。

![Retrofitting%20Fine%20Grain%20Isolation%20in%20the%20Firefox%20R%20296d1bf3188040489d3989aa2afa2d6c/Untitled%204.png](/images/2020-06-23/Untitled%204.png)

### Control flow safety

只能通过sandbox_callback显示注册回调。但是不能让Sandbox任意注册Renderer里的函数，因此需要保证Sandbox只能调用那些被标为tainted的函数。RLBox支持注销回调，这样可以防止Sandbox在非预期时刻调用这个回调函数，具体就是通过C++的RAII语义实现。

## 6. Simplifying migration

迁移步骤如下，可参考[ProcessSandbox的代码示例](https://bitbucket.org/cdisselkoen/sandbox-benchmarking/src/master/main.cpp)理解。

1. 创建沙盒。创建None Sandbox。
2. 分离数据流控制流。通过sandbox_invoke调用库函数，通过sandbox_malloc来分配共享内存。
3. 加固边界。
4. 实施隔离。替换None Sandbox为真正的Sandbox（将None替换为None_DynLib）。

## 7. Implementation

RLBox基于C++的类型系统实现，包括 tainted<T> wrapper。污点的传播通过C++的operator实现。运行时检查用operator[]实现，检查数组的索引是否在Sandbox内存里。指针操作的规则用operator=, operator*, and operator→等实现。

tainted<T>可以自动化对primitive类型、指针类型、函数指针类型、静态数组类型进行封装，但是不能对用户定义的结构体进行封装，因为C++不支持结构体域名的reflection，因此作者写了一个100行的Clang插件解决这个问题。

RLBox API可以支持实现不同的底层Sandbox机制，作者实现了两个Sandbox，一个是Google的基于SFI的NaCl，另一种就是标准的Process sandbox

NaCl/SFI：SFI使用内敛的动态检查来限制库的内存访问只能在限定内存里。

Process sandbox：将库放到被隔离的sandbox进程里运行，系统调用用seccomp-bpf进行限制，用共享内存实现进程间通信。与SFI相比，Process sandbox更容易，不需要修改编译器，性能也甚至可以跟SFI相比。

## 8. Evaluation

实验环境：Intel i7-6700K，64GB RAM，Ubuntu18.04。

**Cross-origin content inclusion：**作者选取了Alexa前500的网站，通过记录网站页面在加载后10秒内发出的跨域请求来评估网站包含的跨域内容。如下图所示。作者得出的结论是93%的站点都加载了至少一个跨域资源，平均为48个。大量的跨域请求也表明<renderer, library, content-origin, content-type>这样的粒度可以显著减少Renderer的攻击面。

![Retrofitting%20Fine%20Grain%20Isolation%20in%20the%20Firefox%20R%20296d1bf3188040489d3989aa2afa2d6c/Untitled%205.png](/images/2020-06-23/Untitled%205.png)

**Sandbox creation：**SFI Sandbox创建开销为1ms，Process Sandbox创建的开销为2ms。

**Control transfer:** 带来的开销，SFI方式为0.22us，process方式为0.47us。 

**RLBox dynamic checks:** 小于1%。

**SFI dynamic checks:** 相比于Process Sandbox来说，SFI本身就有一定的开销，作者测试后发现开销为22%。

**Migrating Firefox to use RLBox：**如下表反映来改造Firefox过程中的人工参与程度以及自动化程度。

![Retrofitting%20Fine%20Grain%20Isolation%20in%20the%20Firefox%20R%20296d1bf3188040489d3989aa2afa2d6c/Untitled%206.png](/images/2020-06-23/Untitled%206.png)

RLBox in Firefox：将libjpeg-turbo 1.4.3，libpng 1.6.3，zlib 1.2.11，libvpx 1.6.1，libtheora 1.2，libvorbis迁移进了沙盒，Firefox版本为57.0.4。

RLBox outside Firefox：应用在Apache server的libmarkdown和Node.js的bcrypt。

## 9. 参考

- [https://github.com/PLSysSec/rlbox-usenix2020-aec](https://github.com/PLSysSec/rlbox-usenix2020-aec)
- [https://wiki.mozilla.org/Community:SummerOfCode19](https://wiki.mozilla.org/Community:SummerOfCode19)
- [https://github.com/shravanrn/mozilla_firefox_nacl/blob/04aaf3534a05bc640346e87902b5460bd38348ee/dom/media/platforms/agnostic/VPXDecoder.cpp](https://github.com/shravanrn/mozilla_firefox_nacl/blob/04aaf3534a05bc640346e87902b5460bd38348ee/dom/media/platforms/agnostic/VPXDecoder.cpp)
- [https://wiki.mozilla.org/Security/Sandbox](https://wiki.mozilla.org/Security/Sandbox)
- [http://phrack.org/issues/69/14.html](http://phrack.org/issues/69/14.html)
- [https://www.chromium.org/developers/design-documents/process-models](https://www.chromium.org/developers/design-documents/process-models)