---
layout: post
title: "The Dangers of Key Reuse: Practical Attacks on IPsec IKE"
date: 2018-09-30 14:38:23 +0800
comments: true
categories: 
---

> 论文链接：https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-felsch.pdf
> 
> 作者：Dennis Felsch, Martin Grothe, and Jörg Schwenk, Ruhr-University Bochum;
> Adam Czubak and Marcin Szymanek, University of Opole
> 
> 会议：the 27th USENIX Security Symposium

## 摘要
在不同版本和不同模式的IKE之间重用密钥对会导致绕过跨协议认证，从而使攻击者可以模仿受害者主机或网络。
Bleichenbacher oracle存在于Cisco、华为、Clavister以及ZyXEL的IKEv1实现中。各平台现已进行了修复。
<!--more-->
## 1 介绍
**IKE**

IKE包括两个阶段:
阶段1：建立初始化的authenticated keying material
阶段2：进一步协商密钥

IKE有两种版本：IKEv1和IKEv2，且均可用。

**认证**
IKEv1：
阶段1可用的认证方法有四种，且均包含DHKE。
IKEv2：
阶段1省略了两种认证方法。

**攻击**
仅针对IKEv1和IKEv2的阶段1，在此阶段冒充IKE设备。一旦在阶段1的攻击成功，攻击者就能与受害设备共享一系列认证的对称密钥，并且可以成功完成阶段2。

**本文的贡献**
- 在四家大型网络设备制造商的IKEv1实施中定义并描述了Bleichenbacher oracles攻击
- 表明oracle的强度足以打破IKEv1和IKEv2中（基于PSK的除外）的所有握手协议变体。
- 证明在某些网络设备中实现的跨协议的密钥重用安全风险很高。
- 完成了对IKEv1以及IKEv2所有变体的评估。

**Responsible Disclosure**
对四大厂商报告了调查结果，各厂商均采取了积极的修复措施。

## 2 IKE（Internet Key Exchange）
负责协商多组加密算法和密钥，在IPsec术语中被称为SA（Security Associations）。

### 2.1 IKEv1
IKEv1包含两个阶段。
阶段1：IKEv1本身建立SA，以便可以加密随后的阶段2的消息。建立了阶段2认证基础的共享对称密钥。
阶段2：协商用于IPsec AH和ESP的若干SA。

**IKEv1 阶段1**
两种模式：main mode和aggressive mode。
四种认证方式：两种基于RSA加密，一种基于签名，一种基于PSK（Pre-Shared Key）。
IKE使用UDP，使用随机值作为cookies（用CI和CR表示）。
IKE协议中阶段1结构：
![](/images/2018-09-30/9lcKg4w.png)
消息m1和m2用于协商不同的加密算法的组合。消息m3和m4用于执行DHKE。双方导出4个对称密钥。m5和m6用于密钥确认。

**IKEv1 阶段2**
基于PSK的认证密钥协商协议。

### 2.2 IKEv1认证方法
IKEv1的阶段1中，有4种认证模式可供选择：
- 数字签名
- PKE（Public Key Encryption）
- RPKE（Revised Public Key Encryption）
- PSKs（Pre-Shared Keys）

**基于签名的认证**
假定每方拥有一个带有效证书的非对称密钥对。将随机数nI和nR作为第三条和第四条消息的辅助信息进行交换。随机数被用作PRF函数的输入，该函数从共享的DH secret中衍生出共享密钥key。每一方需要对他们的MAC值进行签名并交换这些签名和证书（可选项）。

**基于公钥加密(PKE)的认证**
要求双方在事前安全地交换他们的公钥。消息三和四交换随机数和identity作为辅助信息，使用另一方的公钥进行加密来加密辅助信息。双方交换MAC值。
![](/images/2018-09-30/RR0rEgc.png)

**修正的基于公钥加密(RPKE)的认证**
不同之处在于，使用临时对称密钥kel和keR来加密identity，这两个对称密钥由每一方的nonces和cookies衍生而来。

**基于PSK的认证**
通过双方都知道的口令来实现。PSK用来从nl和nR中衍生出k。

### 2.3 IKEv2
IKEv2与IKEv1共享两种认证方法，可以直接应用攻击来模拟IKEv2的阶段1中的IPsec设备。

## 3 Bleichenbacher Oracles
针对RSA PKCS#1.5加密填充的攻击。如果攻击者能够选择明文以0x00 0x02开头，则该攻击是可能的。使用Bleichenbacher攻击可进行签名伪造，且Bleichenbacher攻击可以进行优化。

基本算法：
攻击者窃听到了密码c0，为获得明文m0，攻击者进行查询：
如果m同余于c的d次方mod N，且m以0x00 0x02开头，则O(c)=1；否则O(c)=0。
如果结果为1，则攻击者得知2B<=m<=3B-1，其中B=2的8(lm-2)次方，lm是消息m的字节长度。接着攻击者可以选择值s来计算生成新的候选密文c。
攻击者带上c进行查询，如果结果为0，则增加s并重复前面的步骤。否则，攻击者能得知2B<=m0*s-rN<3B。这使得攻击者能够进一步得到m0的范围：（2B+rN)/s<=m0<(3B+rN)/s。攻击者通过改进s和r的值的猜测并减小包含m0的区间的大小来进行。在某些时候，间隔将包含单个有效值m0。

## 4 攻击大纲

### 4.1 IKEv1中的Bleichenbacher Oracles
替换m3中的cnl，结果有两种情况：
情况0：如果密文不符合PKCS＃1 v1.5规范，则表示出现错误（Cisco，Clavister和ZyXEL）或静默中止（华为）。
情况1：如果密文符合PKCS＃1 v1.5规范，则继续使用消息m4（Cisco和华为）或使用其他消息返回错误通知（Clavister和ZyXEL）。
每次得到情况1的结果，就能进行进一步的Bleichenbacher攻击。

### 4.2 针对PKE和RPKE的Bleichenbacher攻击
![](/images/2018-09-30/Z7tzkxe.png)
1. 攻击者发起与响应者A的基于IKEv1 PKE的密钥交换，并遵守协议，直到收到消息m4。 他们从此消息中提取cnR，并记录公共值cI和cR。 它们还记录nonce nI和自己所选择的私有DHKE密钥x。
2. 攻击者和响应者A的IKE握手保持活动状态，持续最长时间为ttimeout。
3. 攻击者向响应者B发起几个基于PKE的并行密钥交换。
  （1）这些交换中，他们根据协议规范发送和接收前两个消息。
  （2）在消息m3中，根据Bleichenbacher攻击方法，它们包括cnI的修改版本。
  （3）等待直到他们收到m4（情况1），或者他们判定他们的消息不发送出去（超时或接收重复的消息m2）。
4. 在从响应者B收到足够的情况1结果后，攻击者计算nR，计算g的xy次方。
5. 攻击者能够计算MACI并使用密钥ke将消息m5加密发送到响应者A. 因此，他们可以模仿响应者B与响应者A通信.
  值得注意的是，此攻击还可用于对双方执行中间人攻击。

### 4.3 密钥重用
为所有“密码套件系列”维护单独的密钥对和IKE的版本实际上是不可能的，并且通常不被支持。因此，通常的做法是只有一个RSA密钥对用于整个IKE协议族。在这种情况下，协议族的实际安全性取决于其跨密码组和跨版本安全性。

### 4.4 针对基于数字签名的认证的Bleichenbacher攻击
1. 攻击者发起与响应者A的基于IKEv2签名的密钥交换，并遵守协议，直到他们收到消息m2。完成密钥派生。从这些密钥中，他们需要kpI来计算MACI = PRFkpI（IDB），这是要使用响应者B的私钥签名的数据的一部分。
2. 攻击者和响应者A的IKE握手保持活动状态，持续最长时间为ttimeout。
3. 攻击者进行数字签名记为H。
4. 攻击者与响应者B发起几个基于PKE的密钥交换。
  （1）在每个交换中，它们发送和接收前两个消息。
  （2）在消息m3中，它们包括根据Bleichenbacher攻击方法的c的修改版本。
  （3）等到收到答案m4（情况1），或者他们可以可靠地确定不会发送此消息（超时或接收重复的消息m2）。
5. 计算相关数据。
6. 攻击者完成握手。

### 4.5 针对IKEv1主模式的离线字典攻击
如果攻击者处于活动状态并干扰DHKE，则对IKEv1的主模式和PSK的IKEv2也可能发生离线字典攻击。此外，攻击者必须充当响应者，从而等待受害者发起者的连接请求。一旦攻击者主动拦截了这样的IKE会话，他们就会学习加密的MACI值。由于攻击者知道除了PSK之外的所有这些值，就可以对它执行离线字典攻击。

## 5. Cisco IOS中的Bleichenbacher Oracle
思科在IOS中包含PKE身份验证模式。测试使用运行IOS XE的Cisco ASR 1001-X路由器，版本为03.16.02.S，IOS版本为15.5(3)S2。
步骤：
(1)使用默认选项在设备上生成RSA密钥对
(2)使用RSA公钥和测试发起者的IP地址创建了一个对等项。
(3)配置策略，只允许IKEv1和PKE认证。
密文cnI是攻击的目标。该密文与IKEv1握手的消息m3一起发送。向Cisco路由器发送无效密文后，路由器在经过一秒后将消息m2重新发送给发起者。如果路由器成功解密消息，则立即将m4发送给发起者。

### 5.1 测试Oracle的强度
估计需要371,843个请求。

### 5.2 性能局限性
1. Oracle性能限制
2. 攻击性能限制

## 6. Clavister & ZyXEL实现中的Bleichenbacher Oracle
使用版本12.00.06中的虚拟Clavister cOS Core和运行固件版本3.30（AQQ.7）的ZyXEL ZyWALL USG 100。
当发送带有消息m3的无效密文cnI时，会收到一条错误消息，其中只包含16个看似随机的字节。有效的cnI会触发包含字符串“数据长度太大而无法解密私钥”的错误消息。虽然错误消息本身具有误导性），但错误消息的差异显然是Bleichenbacher oracle。

## 7. Huawei Secospace USG2000 series中的Bleichenbacher Oracle
使用了运行固件版本V300R001C10SPC700的华为Secospace USG2205 BSR防火墙。
设置IPsec配置的步骤与Cisco非常相似。在向设备发送带有m3的无效密文后，防火墙不会向发起者发送错误消息，也没有重传。如果防火墙成功处理消息，则将m4发送给发起者。这显然也是一个Bleichenbacher oracle。

### 7.1 测试Oracle的强度
估计一个成功的攻击需要857,571个请求。

## 8.实现Bleichenbacher攻击
1. SA States.汇集SA并跟踪他们的状态。
2. Packet and Network Pool.对于快速攻击，需要一个高效的数据包构建器和分析器。
3. Bleichenbacher Producer and Consumer.实施了两种分配机制（多个和单个间隔），以解决Bleichenbacher攻击中的不同步骤。
4. Cisco Oracle Simulator.

### 8.1 Bleichenbacher IKEv1解密攻击的评价
1. Standard Bleichenbacher.使用1024位公钥和不同的加密nonce执行了990次解密攻击。 平均而言，使用Bleichenbacher原始算法的解密需要303,134个请求。 但是，在78次模拟中，需要不到51,000个请求来解密nonce，因此可能会冒充路由器。
2. Optimized Bleichenbacher.对于优化的Bleichenbacher算法，针对具有不同nonce和1024位密钥的Cisco oracle模拟器执行了200次攻击。
3. Real Cisco Hardware.对于真实硬件的攻击，思科IKEv1状态机的局限性很大。 SA管理的主要障碍是：一旦攻击者与路由器协商数千个SA，其SA处理速度就会变慢。设法对ASR 1001-X路由器进行了成功的解密攻击，大约有19,000个Bleichenbacher请求。

### 8.2 Bleichenbacher IKEv2签名伪造攻击的评价
在时间用完之前发出204,000份Bleichenbacher请求。使用思科oracle模拟器来加速评估。


## 9.针对弱PSK的离线字典攻击
### 9.1 具有弱预共享密钥的IKEv1
在网络上，攻击者等待受害者与响应者发起握手。 如果受害者和响应者已经有活动连接，则攻击者可以通过丢弃已建立连接的所有数据包来强制执行新的握手，这最终将导致新的握手。 在此握手期间，攻击者不会将数据包转发给响应者，而是模拟为响应者（例如通过欺骗其IP地址）。 攻击者充当正常响应者，执行阶段1协议并记录所有交换的消息，直到他们收到消息m5。
使用消息m5，攻击者接收用ke加密的IDI和MACI。 在m5生成的所有值中，攻击者只缺乏IDI和密钥k的知识。 IDI很容易猜到，因为它通常只是发起者的IP地址。 密钥k = PRFPSK（nI，nR）直接来自攻击者想要了解的PSK。
针对此攻击的唯一可用对策是选择能够抵抗字典攻击的加密强PSK。

### 9.2 IKEv2
当前的标准RFC 5996提到，建议将IKEv2与EAP（可扩展认证协议）协议一起使用。但是，在实践中，IKEv2通常在没有EAP的情况下使用。RFC 5996的建议是误导，因为像EAP-MD5或EAP-MSCHAPv2的一些EAP模式，也不会阻止离线字典攻击，他们只是要求攻击者从IKE转向攻击EAP。最后，研究表明，实现仅支持IKEv2和EAP，以便用户远程访问网络。这种结构不包括站点到站点场景，因此仍然容易受到攻击。


## 10. 相关工作
1. IPsec and IKE
2. Bleichenbacher Attacks.
3. Cross Protocol Attacks.

## 11.结论
在给定两个入口点的情况下，可以破坏IPsec的Internet密钥交换（IKE）协议的所有版本和变体。
第一个切入点是弱PSK。 对于所有三种不同的变体，可以使用两种不同的对手进行离线字典攻击：aggressive模式下的IKEv1 PSK可以被被动攻击者破坏，主模式下的IKEv1 PSK和IKEv2 PSK都可以被一个活跃的对手打破，他们扮演一个响应的角色。
第二个切入点是IKEv1 PKE和RPKE变体中的Bleichenbacher oracles。 思科，Clavister，华为和ZyXEL设备中存在这样的oracles。
要对抗这些攻击：只应使用高熵PSK，并且应在所有IKE设备中停用PKE和RPKE模式。
