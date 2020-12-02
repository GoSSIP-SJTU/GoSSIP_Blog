---
layout: post
title: "SoK: SSL and HTTPS: Revisiting past challenges and evaluating certificate trust model enhancements"
date: 2019-11-26 23:42:06 -0500
comments: true
categories: 
---
> 作者：Jeremy Clark and Paul C. van Oorschot 
>
> 单位： School of Computer Science Carleton University, Canada 
>
> 会议：S&P 2013
>
> 资料: [Paper]( https://www.ieee-security.org/TC/SP2013/papers/4977a511.pdf )

---

##  INTRODUCTORY REMARKS 

在本文中，作者提供了SSL / TLS（以下称为TLS）机制的广泛视角，该机制与Web浏览器一起用于保护HTTP流量。

作者重点关注了：

- 对HTTPS安全性进行了不同的分类，涵盖了密码设计和实现，系统软件，操作和人为因素方面
- 总结了未解决的问题和未来的研究方向

<!--more-->
##  BACKGROUND 

**历史目标** a). 机密性和服务器身份验证 b). http+ssl http只需要做小改动

**协议规范** TLS1.0 TLS1.1 TLS1.2 

##  CRYPTO PROTOCOL ISSUES IN HTTPS 

在本节中，我们考虑对与HTTPS安全性相关的TLS协议的攻击。

### 加密基元的弱点

1. 弱加密和签名密钥长度

   - TLS早期版本的密码套件中提供的几种加密功能不再被认为是安全的。

   | 补充： TLS使用的对称加密算法                                 |
   | ------------------------------------------------------------ |
   | TLS1.1使用的算法：RC4 IDEA_CBC DES_CBC 3DES_CBC 3DES_EDE_CBC RC2_CBC                          TLS1.2使用的算法：RC4 3DES_EDE_CBC AES_CBC                                                                                   TLS1.3使用的算法："fewer, better choices" **AES CHACHA20** |

   | 补充：TLS使用的非对称加密算法                                |
   | ------------------------------------------------------------ |
   | TLS1.1使用的算法：RSA DH_DSS DH_anon DHE_DSS KRB5 RSA_EXPORT                                          TLS1.2使用的算法：RSA DH_DSS DH_anon DHE_DSS DH_RSA DHE_RSA                                           TLS1.3使用的算法：DH ECDH && PSK |

2. 弱哈希函数

   要颁发站点证书，CA必须对其哈希签名。 哈希的抗冲突性至关重要：可以构造具有相同摘要的两个有意义的证书的对手可以将CA签名从良性站点证书转移到恶意CA证书。

   | 补充：TLS使用的HASH算法                                      |
   | ------------------------------------------------------------ |
   | TLS1.1使用的算法：MD5 SHA                                                                                                                     TLS1.2使用的算法：MD5 SHA SHA256                                                                                                                   TLS1.3使用的算法：SHA256 SHA384 |

### 实施缺陷和相关攻击

1. PRNG种子

   TLS协议中的许多值都是随机生成的，包括密钥。 这需要强大的伪随机数生成器（PRNG），并使用高熵种子进行播种。 Netscape浏览器（1.22之前的版本）依赖于PRNG实施，该实施带有弱密钥，从而可以预测TLS会话密钥（主密钥）。

2. 远程时序攻击

   漏洞利用是基于时间的旁路攻击。

   | 补充：时序攻击                                               |
   | ------------------------------------------------------------ |
   | 测信道攻击/时序攻击                                                                                                                                                     2003年 -- David Brumley和Dan Boneh：远程定时攻击是可行的。                                                                    2005年 --  D.J. Bernstein公布了一种缓存时序攻击法，破解了一个装载OpenSSL AES加密系统的服务器。                                                                                                                                                             2005年 --  O. Aciic¸mez 指出openssl0.9.7b版本，由于RSA解密过程中使用不同的密钥时间具有可测量的差异，指出了在TLS期间会泄露有关的密钥。                                                                                                             2018年 -- 幽灵漏洞 |

### Oracle攻击  BEAST/LUCKY13/LOGJAM

   1. **RSA Encoding**

      1998年，由Daniel Bleichenbacher发现。基本思想是：通过向服务器发送特制的错误SSL消息，他能够根据服务器发送的错误类型来了解有关密文的微小信息。通过发送足够的消息，他可以学到越来越多的知识，并最终能够使用服务器的私钥完全解密某些内容。（**补充：**USENIX'18  Return Of Bleichenbacher’s Oracle Threat (ROBOT) ）

   2. **CBC Initializing**

      在TLS 1.0和更早版本中，所有块密码均以密码块链接（CBC）模式使用。记录是单独加密的，但是每个记录的初始化向量（除第一个记录外）都设置为等于发送的密文的最后一个块（即以可预测的方式）。会产生BEAST攻击。

   3. **Compression**

      CRIME攻击。此漏洞立足于选择明文攻击配合数据压缩无意间造成的信息泄露。

      它依赖于攻击者能观察浏览器发送的密文的大小，并在同时诱导浏览器发起多个精心设计的到目标网站的连接。攻击者会观察已压缩请求载荷的大小，其中包括两个浏览器只发送到目标网站的私密Cookie，以及攻击者创建的变量内容。当压缩内容的大小降低时，攻击者可以推断注入内容的某些部分与源内容的某些部分匹配，其中包括攻击者想要发掘的私密内容。使用分治法技术可以用较小的尝试次数解读真正秘密的内容，需要恢复的字节数会大幅降低。

   4. **CBC Padding**

      针对CBC填充的Oracle攻击。 

### 协议层面的攻击

1. Ciphersuite Downgrade Attack 密码套件降级攻击

     客户端和服务器在TLS握手期间进行密码套件的协商。（**补充：** logjam）

2. Version Downgrade Attack  版本降级攻击

   握手期间还会协商TLS版本。 为了防止对手先降级到SSL 2.0，然后再降级密码套件，TLS禁止降级到SSL 2.0。 不过，TLS可能仍然容易受到从较新版本降级到较早版本的攻击（例如，从TLS 1.1+降级到TLS 1.0以利用CBC初始化漏洞）。（**补充：** fallbackScsv）

3. Renegotiation Attack  重新协商攻击

   错误版本允许攻击者建立与服务器的连接，重新协商。（**补充**： secure renegotiation）

4. Cross-Protocol Attacks 跨协议攻击

   在跨协议攻击中，攻击者实现客户端在一种协议向解析不同协议的服务器发起事务(transaction)，且在第二种协议中看起来是有效的。使用HTTP/2的ALPN标识符完成TLS握手可以为防护跨协议攻击足够了。


##  TRUST MODEL ISSUES IN HTTPS 

这一节主要关注对CA/B的攻击。

### 证书

DV, domain validated; EV, extended validated; OV, organization validated

*安全问题（证书）*

- 主机名验证（CA） 对于DV，可能会由于DNS毒化或社会工程问题导致颁发错误证书。
- 主机名验证（客户端）大多数浏览器会验证访问域名与证书域名是否相同，但是某些非浏览器软件可能不会。
- 解析攻击 颁发错误证书或浏览器错误解析证书。如，$ \emptyset $问题：bank.com$ \emptyset $evil.com (evil.com/bank.com,/*evil.com)；CA和浏览器还不一致地解释了对象ID,2.5.4.3; B 25.4.003)
- EV降级：
  - 与自动域验证相关的许多问题都被EV证书阻止。
    但是，持有EV证书的站点可以通过带有伪造DV证书的中间人（MITM）攻击而降级为正常的HTTPS。

### 锚信任

根证书即为信任锚 

在专用网络上，尤其在公司环境中，可以将组织的根证书配置为员工计算机上的信任锚。但会存在验证缺陷。

*安全问题（信任锚）*

- CA compromised 攻击弱CA使其错发证书（**补充** CT）
- Compelled Certificates 国家强制措施（哈萨克斯坦证书问题）

### 信任的传递

certificate chain

- basicConstraints CA:True

### 信任维持

主要介绍了证书撤销的问题。**CRL** && **OCSP**（prefer)， 存在CA不响应OCSP的情况或证书到期时间过长。

*安全问题（信任维持）*

- 阻止撤销 撤销信息公布延迟，敌手利用已撤销但还未告知用户的证书，实施中间人攻击。
- 所有权转移 当域名到期或转让时，其证书应该也被撤销。

### 信任的表示和解释

- 浏览器安全提示：https://  or 小锁子
- 浏览器安全警告：如果HTTPS连接由于某些原因失败，浏览器将显示警告。
- 混合内容：上述研究中未考虑的普遍警告。如未使用HTTPS加载的脚本。
- 移动浏览器：通过移动浏览器访问HTTPS网站。
- HTTPS表单提交：http页面，https提交form。

*安全问题（信任的指示和解释）*

- 剥离TLS：利用以上的情景，在用户未知的情况下，将https转为http。
- 欺骗浏览器Chrome：干扰用户看到浏览器的安全提示。
- 发出警告：MITM选择用不受信任的链替换证书，并希望用户单击或忽略警告。

## SECURITY ENHANCEMENTS TO CA/B MODEL

作者将评估了一系列已知提案中最突出的提案。作者不再关注特定工具，而是将每个工具背后的主要概念提炼为一组原语，这些原语可以通过不同方式组合以解决CA / B模型中的安全性问题。如表一所示。

![](/images/2019-11-27\3.PNG)

表一中：

- Security Properties Offered“提供的安全属性”显示了当前HTTPS和CA/B模型不满足但由选定原语（在行中指定）提供的属性。 提供某种增强功能的基元通常会在安全性，隐私，可部署性和可用性方面进行权衡。
- Evaluation of Impact on HTTPS提供了增强的安全性的评估，包括了Security&Privacy、Deployability和Usability。

组织逻辑，当两个原语结合起来的时候：

- Security Properties Offered的或逻辑 - 最强的层次。
- Evaluation of Impact on HTTPS与逻辑 - 最弱的层次。

### A. Security Properties Offered by Primitives 基元提供的安全属性

1. *Detecting Certificate Substitution (Table 1-column A)*  

   1. 测证书是否被替代，以防止MITM攻击，那么属于 *Detects MITM*。如果原语在TOFU（trust on first use）时可能被欺骗，则使用◦表示部分实现。

2. 如果攻击者仅能从客户端方面（如通过DNS毒化或客户端附近的路径拦截）进行MITM攻击，则这种攻击是本地的。如果是可检测的，那么就是 *Detects Local MITM*。

   1. HTTPS经常被用于将客户端的身份信息传递给服务器。一些原语专注于在HTTPS MITM攻击工程中防止用户身份信息被盗取，那么属于 *Protects Client Credential*, ◦的使用同上。
   2. 如果原语在域名的公钥发生改变时报错，则原语的保护属于 *Updatable Pins* 。

3. *Detecting TLS Stripping (Table 1-column B)* MITM对HTTPS POST或GET简单地降级为HTTP

   1. 因为很多对HTTPS的增强都是在客户端与服务器连接时，所以如果能够检测TLS 剥离，则属于 *Detects TLS Stripping*，使用◦表示部分实现。
   2. 阻止（通过强制实施或安全指示符）通过HTTP提交POST请求的原语实现了 *Affirms POST-to-HTTPS*。

4. *PKI Improvements (Table 1-column C)*

   前面介绍了PKI体系的缺陷：缺乏可靠的撤销机制以及对证书链中中间证书的隐藏。当CRLs或OCSP不可用时，仍能检测出证书被撤销，这种情况属于 *Responsive Revocation*，若证书链中间可见，则属于 *CAs Visible*。

### B. Evaluation Criteria for Impact on HTTPS 对HTTPS影响的评估标准

1. *Securtity & Privacy*
   1. 一些原语引入了新的基础结构原语，其中包括了有助于新人决策和在建立HTTPS链接时被查询的实体。如果不引入任何安全实体则满足 *No New Trusted*，如果扩展了已受信任的实体的功能，则为部分实现。
   2. 如果没有引入二任何使用户意识到他们使用HTTPS访问所有站点的新的参与者，则将为 *No New Traceability* 。
   3. 如果原语的引入削弱了某一类安全实体（比如，OCSP的响应），则为 *Reduce Traceability*。
   4. 如果原语引入了新的服务器认证的令牌，比如，OCSP响应、pins等，则为 *No New Auth'n Tokens* 。

2. Deployability
   1. 如果不需要服务器任何的参与或代码的修改，则为 *No Server-side Changes*；如果需要服务器参与，但不需要代码的修改，则为部分实现。 
   2. 如果不需要DNSSEC，则为 *Deployable without DNSSEC* 。
   3. 如果原语没有引入阻止连接完成的额外通信回合，则为 *No Extra Communications* 。
   4. 如果可以支持当前的和之后的HTTPS服务器，则为 *Internet Scalable* 。 
3. Usability
   1. 如果证书对于原语不拒绝具有合法证书的服务器，则为 *No False-Rejects* 。如果原语存在对于正确的证书但存在攻击情况拒绝访问服务器，则为部分实现。
   2. 如果不需要用户的安全问题做出响应，则为 *No New User Descision* 。
   3. 如果原语反应的问题在客户端有信号指示的话，则为 *Status Signalled Completely*。

### C. Summary and Evaluation of Proposed Primitives

1. **Key Pinning (Client History)**

   记住客户端上次访问该服务器时的公钥，若再次访问该服务器使用的公钥与上次访问服务器使用的公钥不同时，则报错。可以防御证书替代攻击，即使该证书没有问题。

2. **Key Pinning (Server)**

   HKPK常采用的策略，服务器可以将它常用的公钥集以及公钥使用的时间放在证书中。TACK策略：不固定证书而固定网站自己设置的经常更换的TACK签名密钥。

3. **Key Pining (Preloaded)**

   浏览器供应商可以在浏览器中包含一个pins列表。 

4. **Key Pining (DNS)**

   有些CA是以DV的方式颁发证书的，但是是否用户自己可以使用DV的方式来验证服务器端的安全性呢？DANE协议建议服务器将其公共密钥固定在DNSSEC记录中，以供客户端进行验证。

5. **Multipath Probing**

   这里引出了众包的概念，由多方决定是否信任这个证书，每一方都会有自己的决策。客观的决策是可以包括时间维度（我在最后一次见过这个包）和空间维度（我看到相似的证书）。多路径探测使用的是基于空间维度的评判标准——它的想法是客户端是否接收该证书的评判标准为该证书是否被网络中与客户端无关的观察者所接受。这样可以预防局部证书替换攻击。如CT。

6. **Channel-bound Credentials**

   通道绑定信任，信任取决于具体的HTTPS链接，即使盗取了站点的可信任证书也无法被信任。将客户端的cookie与原产地证书（origin bound certificate）绑定。

7. **Credential-bound Channels**

   与OBC模式接收客户端自签名证书相反，客户端依据服务器证书是否绑定有客户端的信任来决定是否接收该证书。如预共享密钥机制。

8. **key Agility/Manifest**

   在不在服务器端检测MITM攻击时，可能会由于服务器密钥更新或多个域名共享相同的密钥而导致无法准确的识别是否存在攻击行为。在这种情况下，可采用Key Agility密钥清单，或用旧的证书私钥给新的证书签名，或通过使用主密钥对证书进行修改。

9. **HTTPS-only Pinning  (All Types)** 

   防止HTTPS Stripping攻击。三种形式：HSTS，有服务器端设置；在DNS记录中设置；浏览器内部有记录。                                

10. **Visual Cues for Secure POST**

11. **Browser-stored CRL**

    主流的浏览器通过软件更新手动的撤销高风险证书，Chrome采用了更综合的CRL，这种模式可以透明地更新。

12. **Certificate Status Stapling**

    在“证书状态装订”下，证书持有者会定期获取已签名并带有时间戳的状态报告，并在握手过程中将其包括在证书中。 在HTTPS中，这被定义为TLS扩展，通常称为OCSP装订。但是RFC仅包含了当前服务器的OCSP响应，不包含其信任链的OCSP响应。

13. **Short-lived Certificates**

    在对短期HTTPS证书的最新提议和实施中，按需或成批为服务器颁发了具有四天寿命（大约是OCSP响应的寿命）的证书。 建议与浏览器存储的CRL和密钥固定（服务器）结合使用。

14. **List of Active Certificate**

    每个系统或浏览器都会维护一个信任证书的列表，一般是根证书，如果证书链中的根证书被信任，则该证书被信任。
