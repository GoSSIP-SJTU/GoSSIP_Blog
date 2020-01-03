---
layout: post
title: "Watching You Watch: The Tracking Ecosystem of Over-the-Top TV Streaming Devices"
date: 2020-01-03 01:30:57 -0500
comments: true
categories: 
---
> 作者：Hooman Mohajeri Moghaddam, Gunes Acar, Ben Burgess, Arunesh Mathur, Danny Yuxing Huang, Nick Feamster∗ , Edward W. Felten, Prateek Mittal, Arvind Narayanan
>
> 单位：Princeton University and University of Chicago∗
>
> 出处：CCS' 2019
>
> 链接：https://www.princeton.edu/~pmittal/publications/tv-tracking-ccs19.pdf

<hr/>

1. Abstract & Introduction

这篇文章主要研究对象为一些流媒体设备Over-the-Top(OTT)，OTT是指诸如Roku TV和Amazon Fire TV这样的设备。OTT设备提供了多频道电视订阅服务，并且通过广告来盈利。作者为了研究这类平台的隐私问题，开发了一个系统自动地下载OTT应用，并与之交互。作者使用爬虫在Roku和Amazon Fire TV上访问了2000多个频道，并且在69％的Roku频道和89％的Amazon Fire TV频道中都存在广告跟踪器的流量。作者还发现了有时会通过未加密的连接收集和传输唯一标识符（例如设备ID，序列号，WiFi MAC地址和SSID）。最后作者证明了这些设备上可用的对策（例如限制广告跟踪选项和广告屏蔽）实际上是无效的。

<!--more-->

作者在分析过程中有以下两个挑战：

- 自动化地与OTT设备进行交互以触发这些应用，并获得应用输出的反馈。作者开发了一个使用音频和电视屏显像素信息来判断电视上视频播放状态的系统。
- 由于OTT设备不像Web浏览器和移动设备一样可以安装根证书拦截HTTPS流量，作者通过修改了mitmproxy，将自己的TLS证书部署到设备上，并使用frida绕过ssl pinning以来拦截和解密HTTPS流量。

贡献如下：


作者在分析过程中有以下两个挑战：

- 自动化地与OTT设备进行交互以触发这些应用，并获得应用输出的反馈。作者开发了一个使用音频和电视屏显像素信息来判断电视上视频播放状态的系统。
- 由于OTT设备不像Web浏览器和移动设备一样可以安装根证书拦截HTTPS流量，作者通过修改了mitmproxy，将自己的TLS证书部署到设备上，并使用frida绕过ssl pinning以来拦截和解密HTTPS流量。

贡献如下：

- 进行了OTT流媒体频道隐私实践的首次大规模研究；
- 实现了第一个可以自动完成OTT频道互动和网络活动拦截的系统；
- 发现了两个不同平台存在分别有69%和89%的广告追踪器流量；
- 通过分析OTT设备的本地远程控制API，发现了一个漏洞，该漏洞使恶意Web脚本能够获取用户敏感信息。

###2. Background

作者研究了两种最受欢迎的OTT流媒体设备：Roku和Amazon Fire TV，因为这两种设备的全球市场份额的59％至65％。这两种设备均需要用户通过HDMI连接到显示器（例如电视）来完成使用。用户可以从Roku商店和Amazon Fire TV商店下载并在其设备上安装频道。 

Roku流媒体设备运行由Roku公司开发的专有操作系统。Roku通道经过打包，签名和加密，以确保源代码的机密性，只有Roku设备才能解密频道。作为一种隐私选项，Roku允许用户“限制广告跟踪”以禁用用于目标广告的标识符，Roku的开发者文档指出，启用此选项后，渠道不应使用从设备收集的数据来投放个性化广告。 Amazon Fire TV设备运行Android操作系统的自定义版本。 

Amazon Fire TV频道以Android应用程序包（APK）格式打包，并且与大多数Android设备一样，开发人员可以使用adb工具与Fire TV设备进行交互。Amazon也允许用户“禁用基于兴趣的广告”并限制行为分析和定位。

### 3. Smart Crawler and Data Collection

作者在这部分描述了所收集的数据信息。

#### 3.1频道信息

作者收集了两种频道信息，一种是Roku频道，另一种是Amazon Fire TV频道。收集时间是2019年5月。Roku频道列表包括23个类别的8,660个Roku频道。Amazon Fire TV频道列表包括扩29个类别的6,782个频道的列表。作者收集信息包含类别中的频道列表以及每个频道的元数据信息，包括其ID，说明，受欢迎程度以及开发人员的身份。



#### 3.2 爬虫信息

爬虫的架构如图所示：

![image-20200103011153688](/Users/mxgc/Dropbox/Paper Reading/Watching You Watch The Tracking Ecosystem of OTT TV Streaming Devices/assets/image-20200103011153688.png)

有台式机，电视显示器，HDMI分流器和捕获卡以及OTT设备。其中台式机充当WiFi接入点（AP），其无线网络接口桥接到Internet。OTT设备连接到此AP，这样可以捕获OTT设备的网络流量。 OTT设备通过HDMI捕获卡将其视频输出到电视显示器和台式机。通过显示器检查爬虫的行为，并使用台式机上保存的屏幕截图来直观地验证OTT设备的输出状态以进行进一步调试。最后，电视显示器的音频输出连接到台式机的音频输入并保存到文件中。

在交互时，对于Amazon Fire TV使用adb进行操作，对于Roku使用Web API（http://ROKU_DEVICE_IP_ADDRESS:8060/keydown/left）进行操作。操作过程为：从频道列表一次操作一个频道，使用爬虫将安装一个频道，启动该频道，并与之交互以播放视频，然后尝试解密各种网络和应用级的数据，最后卸载频道。

![image-20200103015443300](/Users/mxgc/Dropbox/Paper Reading/Watching You Watch The Tracking Ecosystem of OTT TV Streaming Devices/assets/image-20200103015443300.png)

在收集数据时，会使用mitmproxy解密TLS流量，对于Roku设备上无法安装证书的情况下，尝试在Desktop机器上修改Roku设备的DNS信息进行中间人攻击，如果一些域名TLS拦截失败，则加入黑名单不进行劫持。对于Amazon的设备，则是使用安装证书来完成HTTPS流量的解密，对于使用了SSL pinning的频道，使用Frida进行bypass。得到的数据如下：

![image-20200103013550963](/Users/mxgc/Dropbox/Paper Reading/Watching You Watch The Tracking Ecosystem of OTT TV Streaming Devices/assets/image-20200103013550963.png)

###4. PROCESSING CRAWL DATA

#### 4.1 Data Files

对于每个频道会生成许多文件：

- pcap文件：每个频道的流量
- ssl密钥：包含了pre master secret的密钥文件
- 时间戳和事件：爬虫进行的操作，诸如频道安装，频道启动，频道卸载和任何按键（使用远程控制API）
- TLS域名列表：记录了成功和未成功拦截的TLS端点列表，以及相应的域名
- 屏幕截图：在每个频道中捕获的屏幕截图，使用这些屏幕截图得知交互时屏幕的输出
- 音频记录：记录了每个频道播放的所有音频，使用这些文件来检测视频播放

#### 4.2 Data Processing

在这一部分，作者对数据进行处理，主要处理的数据为网络数据。首先将IP地址映射到主机名，分为三种情况：

1. 首先检查是否存在具有匹配TCP流索引的HTTP请求。如果有，则分配Host标头请求。
2. 搜索通过此TCP连接发送的TLS握手，并使用其TLS服务器的SNI字段来确定主机名
3. 如果1、2均失败，使用在此频道搜寻期间进行的DNS查询来确定与服务器IP地址相对应的主机名

图4说明了作者的数据处理流程：

![image-20200103013535211](/Users/mxgc/Dropbox/Paper Reading/Watching You Watch The Tracking Ecosystem of OTT TV Streaming Devices/assets/image-20200103013535211.png)

最后，作者使用了频道meta数据来检索频道ID，类别和等级，以标记观察到的网络流量。



### 5 FINDINGS

作者认为他们的黑名单策略不会影响频道中视频的播放，只是一些需要用户登录才能播放的频道则会影响视频的播放。

首先是广告跟踪器，表2列出了Roku-Top1K-MITM和FireTV-Top1K-MITM中所有频道上出现频率最高的10个跟踪器。其中Amazon Fire TV频道中，亚马逊自己的广告跟踪域名amazon-adsystem.com最多。

![image-20200103013605101](/Users/mxgc/Dropbox/Paper Reading/Watching You Watch The Tracking Ecosystem of OTT TV Streaming Devices/assets/image-20200103013605101.png)

然后是跟踪器的分布，如图5所示。其中总体而言，在Roku和Amazon Fire TV频道中，作者观察到播放视频（橙色点）的频道比不播放视频（蓝色点）的频道包含更多的跟踪器。

![image-20200103014932435](/Users/mxgc/Dropbox/Paper Reading/Watching You Watch The Tracking Ecosystem of OTT TV Streaming Devices/assets/image-20200103014932435.png)



再然后是标识符和信息泄露问题，作者使用Englehardt等人描述的方法搜索了各种编码和散列组合，以检测URL和Base64编码的标识符和哈希ID。在Roku上，作者发现6142个请求中的4452个（占73％）包含两个唯一ID（广告ID，序列号）之一被标记为跟踪器。在Amazon的8433个中，有3,427个（41％）的唯一标识符以明文形式发送。

前面所叙述的跟踪器是使用广告屏蔽列表去做的，作者接下来去检测未知的跟踪器。例如，作者检查了一些跟踪器是否使用了视频标题以获取用户的播放信息。这种跟踪器作者通过以下方法检测，从检测到视频播放的频道子集中随机选择了每个平台上的100个频道。然后手动查看了每个频道菜单中的屏幕截图，以确定视频的标题。如果某个频道具有菜单中显示的通用标题（例如“ Live News”），则将查看视频播放本身的屏幕截图以确定是不是同一个标题。作者在下面列出泄露频道名称、视频标题和跟踪域名的信息：

![image-20200103015710104](/Users/mxgc/Dropbox/Paper Reading/Watching You Watch The Tracking Ecosystem of OTT TV Streaming Devices/assets/image-20200103015710104.png)

最后作者还发现了一些远程控制API的漏洞，作者分析了Roku和Fire的本地API，发现Roku的API允许发送命令以安装/卸载/更改频道并收集设备信息。 API返回的设备信息包括已安装频道的完整列表，唯一的设备标识符（例如MAC地址，广告ID，序列号）和设备所连接的无线网络的SSID。
- 进行了OTT流媒体频道隐私实践的首次大规模研究；
- 实现了第一个可以自动完成OTT频道互动和网络活动拦截的系统；
- 发现了两个不同平台存在分别有69%和89%的广告追踪器流量；
- 通过分析OTT设备的本地远程控制API，发现了一个漏洞，该漏洞使恶意Web脚本能够获取用户敏感信息。

###2. Background

作者研究了两种最受欢迎的OTT流媒体设备：Roku和Amazon Fire TV，因为这两种设备的全球市场份额的59％至65％。这两种设备均需要用户通过HDMI连接到显示器（例如电视）来完成使用。用户可以从Roku商店和Amazon Fire TV商店下载并在其设备上安装频道。 

Roku流媒体设备运行由Roku公司开发的专有操作系统。Roku通道经过打包，签名和加密，以确保源代码的机密性，只有Roku设备才能解密频道。作为一种隐私选项，Roku允许用户“限制广告跟踪”以禁用用于目标广告的标识符，Roku的开发者文档指出，启用此选项后，渠道不应使用从设备收集的数据来投放个性化广告。 Amazon Fire TV设备运行Android操作系统的自定义版本。 

Amazon Fire TV频道以Android应用程序包（APK）格式打包，并且与大多数Android设备一样，开发人员可以使用adb工具与Fire TV设备进行交互。Amazon也允许用户“禁用基于兴趣的广告”并限制行为分析和定位。

### 3. Smart Crawler and Data Collection

作者在这部分描述了所收集的数据信息。

#### 3.1频道信息

作者收集了两种频道信息，一种是Roku频道，另一种是Amazon Fire TV频道。收集时间是2019年5月。Roku频道列表包括23个类别的8,660个Roku频道。Amazon Fire TV频道列表包括扩29个类别的6,782个频道的列表。作者收集信息包含类别中的频道列表以及每个频道的元数据信息，包括其ID，说明，受欢迎程度以及开发人员的身份。



#### 3.2 爬虫信息

爬虫的架构如图所示：

![image-20200103011153688](/images/2020-01-03/image-20200103011153688.png)

有台式机，电视显示器，HDMI分流器和捕获卡以及OTT设备。其中台式机充当WiFi接入点（AP），其无线网络接口桥接到Internet。OTT设备连接到此AP，这样可以捕获OTT设备的网络流量。 OTT设备通过HDMI捕获卡将其视频输出到电视显示器和台式机。通过显示器检查爬虫的行为，并使用台式机上保存的屏幕截图来直观地验证OTT设备的输出状态以进行进一步调试。最后，电视显示器的音频输出连接到台式机的音频输入并保存到文件中。

在交互时，对于Amazon Fire TV使用adb进行操作，对于Roku使用Web API（http://ROKU_DEVICE_IP_ADDRESS:8060/keydown/left）进行操作。操作过程为：从频道列表一次操作一个频道，使用爬虫将安装一个频道，启动该频道，并与之交互以播放视频，然后尝试解密各种网络和应用级的数据，最后卸载频道。

![image-20200103015443300](/images/2020-01-03/image-20200103015443300.png)

在收集数据时，会使用mitmproxy解密TLS流量，对于Roku设备上无法安装证书的情况下，尝试在Desktop机器上修改Roku设备的DNS信息进行中间人攻击，如果一些域名TLS拦截失败，则加入黑名单不进行劫持。对于Amazon的设备，则是使用安装证书来完成HTTPS流量的解密，对于使用了SSL pinning的频道，使用Frida进行bypass。得到的数据如下：

![image-20200103013550963](/images/2020-01-03/image-20200103013550963.png)

###4. PROCESSING CRAWL DATA

#### 4.1 Data Files

对于每个频道会生成许多文件：

- pcap文件：每个频道的流量
- ssl密钥：包含了pre master secret的密钥文件
- 时间戳和事件：爬虫进行的操作，诸如频道安装，频道启动，频道卸载和任何按键（使用远程控制API）
- TLS域名列表：记录了成功和未成功拦截的TLS端点列表，以及相应的域名
- 屏幕截图：在每个频道中捕获的屏幕截图，使用这些屏幕截图得知交互时屏幕的输出
- 音频记录：记录了每个频道播放的所有音频，使用这些文件来检测视频播放

#### 4.2 Data Processing

在这一部分，作者对数据进行处理，主要处理的数据为网络数据。首先将IP地址映射到主机名，分为三种情况：

1. 首先检查是否存在具有匹配TCP流索引的HTTP请求。如果有，则分配Host标头请求。
2. 搜索通过此TCP连接发送的TLS握手，并使用其TLS服务器的SNI字段来确定主机名
3. 如果1、2均失败，使用在此频道搜寻期间进行的DNS查询来确定与服务器IP地址相对应的主机名

图4说明了作者的数据处理流程：

![image-20200103013535211](/images/2020-01-03/image-20200103013535211.png)

最后，作者使用了频道meta数据来检索频道ID，类别和等级，以标记观察到的网络流量。



### 5 FINDINGS

作者认为他们的黑名单策略不会影响频道中视频的播放，只是一些需要用户登录才能播放的频道则会影响视频的播放。

首先是广告跟踪器，表2列出了Roku-Top1K-MITM和FireTV-Top1K-MITM中所有频道上出现频率最高的10个跟踪器。其中Amazon Fire TV频道中，亚马逊自己的广告跟踪域名amazon-adsystem.com最多。

![image-20200103013605101](/images/2020-01-03/image-20200103013605101.png)

然后是跟踪器的分布，如图5所示。其中总体而言，在Roku和Amazon Fire TV频道中，作者观察到播放视频（橙色点）的频道比不播放视频（蓝色点）的频道包含更多的跟踪器。

![image-20200103014932435](/images/2020-01-03/image-20200103014932435.png)



再然后是标识符和信息泄露问题，作者使用Englehardt等人描述的方法搜索了各种编码和散列组合，以检测URL和Base64编码的标识符和哈希ID。在Roku上，作者发现6142个请求中的4452个（占73％）包含两个唯一ID（广告ID，序列号）之一被标记为跟踪器。在Amazon的8433个中，有3,427个（41％）的唯一标识符以明文形式发送。

前面所叙述的跟踪器是使用广告屏蔽列表去做的，作者接下来去检测未知的跟踪器。例如，作者检查了一些跟踪器是否使用了视频标题以获取用户的播放信息。这种跟踪器作者通过以下方法检测，从检测到视频播放的频道子集中随机选择了每个平台上的100个频道。然后手动查看了每个频道菜单中的屏幕截图，以确定视频的标题。如果某个频道具有菜单中显示的通用标题（例如“ Live News”），则将查看视频播放本身的屏幕截图以确定是不是同一个标题。作者在下面列出泄露频道名称、视频标题和跟踪域名的信息：

![image-20200103015710104](/images/2020-01-03/image-20200103015710104.png)

最后作者还发现了一些远程控制API的漏洞，作者分析了Roku和Fire的本地API，发现Roku的API允许发送命令以安装/卸载/更改频道并收集设备信息。 API返回的设备信息包括已安装频道的完整列表，唯一的设备标识符（例如MAC地址，广告ID，序列号）和设备所连接的无线网络的SSID。
