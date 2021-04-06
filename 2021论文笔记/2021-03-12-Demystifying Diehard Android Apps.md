# Demystifying Diehard Android Apps

> 作者：Hao Zhou，Haoyu Wang，Yajin Zhou， Xiapu Luo，Yutian Tang，Lei Xue，Ting Wang
>
> 单位：The Hong Kong Polytechnic University，Beijing University of Posts and Telecommunications， Zhejiang University，ShanghaiTech University，Pennsylvania State University
>
> 会议：ASE 2020
>
> 链接：[Demystifying Diehard Android Apps](http://yajin.org/papers/diehard_ase20.pdf)

## 简介

Android采用了一些资源管理回收的机制，会清理后台进程。一些App，为了提供及时服务（如，发送提示）或者获取收益（如，推送广告），会调用可以防止自己被清理方法，这些方法被称为diehard (顽固) method。顽固方法可以被粗略的分为两类：（1）使App保持活动状态； （2）唤醒App。

本篇文章是一个对顽固方法进行系统性研究的工作，总共发现了12个顽固方法。作者开发了一个工具DiehardDetector，可以对使用顽固方法的应用进行检测。并且在超过80k个应用中，检测出了21%的应用具有顽固行为。

<!-- more -->

## 背景

Android框架使用两个数据结构ProcessRecord和TaskRecord来管理应用进程。 

PrecessRecord是ActivityManagerService用来标记运行中应用进程的信息，其成员变量`curAdj`（或`setAdj`）和`curSchedGroup` (或者 `setSchedGroup`)保存了应用进程的属性值。

TaskRecord用于标志activity Task。在正常情况下，最近访问的应用程序的TaskRecord对象对应于“最近任务”列表中的项。 因此，`TaskRecord`对象将最近访问的应用程序的activity task与最近任务项链接在一起。

## 杀死App进程的方法

### 寻找Process-killing方法

Android有两种基本的方法可以结束App进程：1）调用ProcessRecord里的kill API；2）使用libc.so里的kill函数。

### 应用进程优先级

Android使用out-of-memory调节和线程调度组决定应用进程的级别。

**OOM调节。** 具有OOM调节分数越小的应用优先级越高，越不容易被杀死。

**线程调度组。** AMS使用线程调度组来管理一组应用进程的生存。同样的优先级的应用属于一个组。`SECHD_GROUP_TOP_APP`为前台应用优先级更高。`SECHD_GROUP_BACKGROUND`为后台应用，优先级低。

![](/images/2021-03-12-1/upload_c33b08de8c13f11c69d760c7ac99d469.png)


### 终止进程方法

- Removing Recent-Task Item (RRT)

用户在最近任务列表中结束任务。

RRT不能结束不在最近任务列表中的项。也不能杀死具有多个任务或者非`SCHED_GROUP_BACKGROUND`的进程。

- 强制结束app（FSA）。

用户强制结束App或者调试器使用`am force-stop`强制结束App。

可以杀死一般app的所有进程，但不能杀死instant app。

![](/images/2021-03-12-1/upload_b0d493f4d81b0928bfce5dd1c78db7e8.png)


- 杀死后台进程（KBP）

具有`KILL_BACKGROUND_PROCESSES`权限的应用或者调试器使用`am kill`可以杀死后台进程。但是不能杀死OMM值小于`SERVICE_ADJ`的进程。

- 杀死所有的后台进程（KAP）

调试器可以使用`am kill-all`杀死所有的后台进程。但是OMM值小于`CACHED_APP_MIN_ADJ`的进程不会被杀死。

- 杀死特定进程（KSP）

应用和调试器可以分别使用native函数kill和kill命令根据PID杀死进程。

- Low Memory Killer (LMK)

当可用RAM空间不足时，LMK守护进程会杀死一些应用进程。但是这种方法很少杀死高优先级的进程（OOM值低）。

## 保持应用活跃



### 修改最近任务项

可以用于绕过RRT（移除最近任务）

- 隐藏最近任务项（HTI）
    在manifest文件或者代码里设置excludeFromRecents属性为True可以不在最近任务项里隐藏任务。

- 产生多个最近任务项（PMI）

### 提高App进程优先级

可以避免RRT，KBP，KAP，LMK。

- 保持前台activity（HFA）

  可以避免KBP,KAP。注册一个receiver，用于接收system/app的广播，比如接收到广播`ACTION_SCREEN_OFF`时起一个前台activity。

- 托管前台service（HFS）

  前台service对应的OOM调节是`PERCEPTIBLE_APP_ADJ`，要比`SERVICE_ADJ` 和`CACHED_APP_MIN_ADJ`小，不会被KBP和KAP杀死，但是创建时会告知给用户。

- 创建覆盖窗口（COW）

  覆盖窗口在所有app的上面，调度组为`SCHED_GROUP_DEFAULT`，OOM调度值为`PERCEPTIBLE_APP_ADJ`。
  Android 8.0之前把window的类型设置为TYPE_PHONE, TYPE_SYSTEM_ALERT, TYPE_SYSTEM_ERROR 或者TYPE_TOAST，Android 8.0及之后使用 TYPE_APPLICATION_OVERLAY创建覆盖窗口。

- 绑定运行服务（BRS）

  如果service是在单独的进程里运行的，由于service一般不会使用用户可感知的组件（如UI），进程的调度值大于`PERCEPTIBLE_APP_ADJ`。但如果它的客户端进程调度值较小，会取最小的客户端的进程调度值，并且把调度组设置为`SCHED_GROUP_DEFAULT`。所以可以使用bindService绑定自己的service进程，防止被RRT, KBP, 和KAP杀掉。

- 请求发布的Content Provider（ACP）

  同service进程的处理方式。

<!-- ![img](fig/tab2.png) -->

![](/images/2021-03-12-1/upload_46dcca46f400dcb939e6b73bf76f1985.png)


## 唤醒应用



### 使用应用组件

- 构造Sticky Service（CSS）

  粘性service当被RRT, KBP, KAP, KSP, 或LMK杀掉后（FSA杀掉的会忽略），Android会重启它。
  设定回调函数onStartCommand的返回值为`START_STICKY_COMPATIBILITY`, `START_STICKY`和`START_REDELIVER_INTENT`，指定为前两个常量可多次重启，最后一个只重启一次。

  

### 借助系统功能

- 监控系统广播（MSB）

 通过在manifest里注册广播接收器（使用` registerReceiver`动态注册的接收器一旦被停止后不能接收广播），一旦接收到某个事件重启。
同样不适用于FSA, FSA杀掉的进程只能接收带有 `FLAG_INCLUDE_STOPPED_PACKAGES`标志的事件，而系统广播不带有这个标志。
  并且在Android8.0以后，静态申明的receiver不允许接收隐式的广播（大部分系统广播），只有表4的部分广播不受这个限制。

![](/images/2021-03-12-1/upload_796bf2b202f3851b716ef8a3712733f3.png)


- 借助Alarm Sevice（LAS）

 AlarmManager允许应用重复启动组件，被RRT, KBP, KAP, KSP和LMK杀掉的方法可以被唤醒。FSA不行，被FSA杀死的应用会从Alarm Service里移除。

- 使用Job Scheduling Service（UJS）

Android 5.0以后，应用可以向`JobSechedulerService`提交`JobInfo`对象申请周期性工作，被RRT, KBP, KAP, KSP或者LMK杀死的应用可以被唤醒，但FSA同样不行。
Android 8.0+这个周期被限定最短15分钟，但是Android 10.0以前有一个漏洞，系统只在`setPeriodic`检查了时间间隔，但没有在服务端检查，可以篡改这个时间间隔。
![](/images/2021-03-12-1/upload_f5d053a48cd3d1118b7a9afd580c3bf3.png)


此外，instant app由于无法被`JobSechedulerService`通过包名和类名识别也无法被唤醒。

### 借助第三方应用

- 监控应用广播（MAB）

其他应用的广播可以唤醒被包括FSA的所有方法杀死的应用。三方应用的广播需要带上两个标志`FLAG_RECEIVER_INCLUDE_BACKGROUND`（规避Android 8.0后不能接收隐式广播的限制）和`FLAG_INCLUDE_STOPPED_PACKAGES`（FSA杀死的应用也可以接收广播）。

## 检测顽固应用

<!-- ![img](fig/fig4.png) -->

![](/images/2021-03-12-1/upload_c71e8ef7b9d30dabf5362be2d6c78341.png)


### Manifest Parser

- 获得导出组件和接收隐式intent的组件集合$S_{exported}$

- 解析intent-filters内容，获得监听的广播集合 $S_{syscast}$ and $S_{appcast}$

- 特殊的应用组件。有process标志的服务或者content provider运行在独立的进程（$S_{process}$）。有taskAffinity属性的activity运行在独立的Task（$S_{affinity}$）。有excludeFromRecents=true的activity的task不出现在最近任务里（$S_{recents}$）。

- 声明权限$S_{permission}$

### 分析Bytecode

- Call Graph。使用FlowDroid生成call graph，并通过筛选可启动的应用组件修剪其中不可达的边。可启动的应用组件包括3类：1）可导出的应用组件；2）被可导出组件调用的组件；3）被已知可启动组件调用的组件。通过定位表6中的API和Intent的参数获得调用和被调用的组件】。

<!-- ![img](fig/tab6.png) -->

![](/images/2021-03-12-1/upload_2e86a0e34eff81192a0fb0906058777d.png)


- Def-use链。
    - 找overlay窗口检测COW: Window的setAttributes方法的a参数或Window的setType方法的type参数
    - 找不会ubound的服务检测BRS: ServiceConnection对象，包括通过bindService API或unbindService API传递的对象。
    - 找不会被释放或关闭的ContentProvider检测ACP：ContentProviderClient对象，包括AcquisitionContentProviderClient的返回对象或者是ContentProviderClient的release/close方法调用的基础对象。
    - 识别PMI，CSS和MAB需要找使用特殊的Intent标志启动的组件：Intent对象。将使用`FLAG_ACTIVITY_NEW_TASK`标志启动的activity存储到集合$S_{task}$中，将由startService启动的服务保存到集合$S_{startService}$中，并将发送的广播带有标志`FLAG_RECEIVER_INCLUDE_BACKGROUND`和`FLAG_INCLUDE_STOPPED_PACKAGE`放入集合$S_{sndcast}$中。

- 可启动的组件集合$S_{launchable}$
- 可启动组件的可调用的方法集合$S_{callable}$

### 检测顽固方法

<!-- ![img](fig/tab5.png) -->

![](/images/2021-03-12-1/upload_aead763c75610cfe7d04f64ad2291c0a.png)


## 评估

### RQ1： 在Android5.1到10.0的版本上这些顽固方法都可行吗？

除了10.0上UJS不可行，5.1到10.0上其他方法都可行。

### RQ2：DiehardDetector的表现


作者从F-Droid上下载了2k+个应用进行测试。DiehardDetector成功分析了2,071个应用程序（9个在生成call graph时间过长失败），发现其中632个（占30.52％）使用了顽固方法，精度和召回率分别为100％和94.70％。

作者在检测结果中随机分别挑选了250个检测出顽固方法的应用和250个没有检测出顽固方法的应用。
- 有顽固方法的应用没有发现任何误报
- 普通应用14个漏报，FlowDroid在构造调用图时引发异常

表中将检测结果（#app'）与未删除无效调用图边的检测结果（#app）进行了比较(Δapp＝＃app'-＃app)。如果不删除无效边，将导致53个误报。

<!-- ![img](fig/tab7.png) -->

![](/images/2021-03-12-1/upload_447cdf1c42835203f17c7d0445a1031a.png)


### RQ3：顽固应用的普遍性

作者在Google Play上爬取了80k个应用。

DiehardDetector分析应用程序的平均时间为45秒。表8列出了总体结果，其中Ratio是具有特定顽固方法的应用的比率。数据集中有17,410个（约21％）的Google Play应用使用了顽固方法，即顽固应用在应用生态系统中非常普遍。此外，MSB，LAS，CSS，HFS和HTI是最流行的顽固方法，大约70％的顽固应用程序实现了MSB。在data-2k和data-80k中未找到ACP和MAB。因此这是作者提出的新方法。

![](/images/2021-03-12-1/upload_7498c61af53f385b1385bf796d23dd90.png)


由于一个顽固的应用程序可能采用不止一种顽固方法，因此作者还分析了每个顽固的应用程序中使用的顽固方法的数量，结果如表9所示。超过35％的顽固的应用程序采用不止一种顽固的方法方法，但很少有顽固的应用程序采用四种以上的顽固方法。

![](/images/2021-03-12-1/upload_43cd91dc52ea6acb3c25144b4fac94c5.png)


表10是顽固应用程序在不同应用类别中的分布。Personalization类别中超过40％的应用程序使用顽固方法。此外，包括通讯，新闻杂志和工具在内的类别拥有30％以上的顽固应用。

![](/images/2021-03-12-1/upload_f70816277eba7cffeb1f26b62afcdb0e.png)
