---
layout: post
title: eMRTD电子机读旅行证件阅读操作
time: 2022年2月12日 星期六
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: SmartCard
tags: [cryptography, smartcard]
excerpt_separator: ""
---

电子护照eMRTDs定义在ICAO Doc9303号文件《机读旅行证件》（Machine Readable Travel Documents）的第11部分**机读旅行证件安全机制(Security Mechanisms for MRTDS)**中。

我们常见的出入境证件各国的电子护照，卡片式的港澳通行证，还有一些国家地区的电子身份证都是按照eMRTDs的标准实现的。

# 保护电子数据

访问电子护照包括以下步骤
- 获得eMRTD的非接触访问权限
- 数据认证
- 芯片认证
- 其他访问控制机制
- 读取数据
不同的步骤有可以使用的不同协议，eMRTD的具体配置有签发机构选定。ICAO给出了选择的建议。

*BASELINE SECURITY METHOD*  

|方法|非接触|查验系统|优点|说明|
|----|----|----|----|----|
|被动认证|M|M|证明 $SO_D$ 和LDS的内容真实且未被修改|不能防止完全复制或者芯片被替换；<br/>不能防止未经授权的访问；<br/>不能防止非法浏览|

*ADVANCED SECURITY METHODS*  

|方法|非接触|查验系统|优点|说明|
|----|----|----|----|----|
|传统MRZ(OCR-B)和基于芯片的MRZ（LDS）的比较|n/a|o|证明芯片内容和物理的eMRTD匹配|增加少许的复杂性<br/>不能防止对芯片和传统证件的完全复制|
|主动认证<br>芯片认证|O|o|防止拷贝$SO_D$，并证明他是从真实的芯片中读取的<br/>证明非接触IC没有被替换|不能防止未经授权的访问<br/>增加复杂度|
|基本访问控制<br/>*Basic Access Control*|r/c|m|防止非法浏览和误用<br/>防止窃听eMRTD和查验系统之间的通信（用于建立加密会话的信道时）|不能防止完全复制或IC被替换（还需要复制传统证件）<br/>增加复杂性|
|口令认证链接确立协议<br/>*Password Authenticated Connection Establishment*|R|r|同上|同上|
|扩展访问控制|o|o|防止对其他生物特征进行非授权的访问<br/>防止非法浏览其他生物特征|需要额外的密钥管理<br/>不能防止完全复制或IC替换<br/>增加复杂性|
|数据加密|o|o|保护其他生物特征的安全<br/>不需要芯片处理器|需要复杂的解密密钥管理<br/>不能防止完全拷贝或IC替换<br/>增加复杂性|

*m=Required,r=Recommended,o=Optional,c=Conditional,n/a=not applicable*

# 访问非接触IC

MRTD增加非接触IC，如果没有访问控制会带来两种可能的攻击：
- 在未经授权的情况下对存储在非接触IC中的数据进行阅读
- 非接触IC和Reader之间的未加密通信会被窃听

签发机构应该选择实施一种芯片范文控制机制，即证件持有者知道存储在非接触IC中的数据在被安全读取的一种访问控制机制。

查验系统必须首先得到这方面的信息才能够阅读非接触IC，信息必须从eMRTD中以光学或者目视的方式获得。在不能进行机器阅读信息时，查验人员还可以以人工的方式将该信息输入到查验系统。
假定数据页上的信息不能从没有经过查验的证件上得到，可以认为eMRTD时在持证者之情的情况下交送查验的。
**举个例子，比如必须知道MRZ才能获得护照非接触IC的访问权限，你护照的MRZ信息必须在你把护照交给查验人员后，查验人员可才能获得MRZ的数据，因此如果查验系统知道了你MRZ，那么就认为你知道要接受证件查验了**

## 合规的设置

下面的配置符合ICAO的规范：
- 不实施芯片访问控制的eMRTD芯片
- 仅实施BAC的eMRTD芯片
- 实施BAC和PACE的eMRTD芯片
- 从2018年1月起，仅实施PACE的eMRTD芯片
*为了保证全球兼容，各国在2017年12月31日前不能在未实施BAC的情况下仅实施PACE。如果eMRTD芯片只支持PACE，查验系统应实施和使用PACE*

## 芯片的访问过程
### 1 读取EF.CardAccess
如果eMRTD支持PACE，eMRTD芯片在文件EF.CardAcess中必须提供用于PACE的参数。
如果EF.CardAcess可用，查验系统应读取文件EF.CardAcess来确定eMRTD芯片支持的参数（对称密码、密钥协商算法、域参数和映射），查验系统可以选择这些参数中的任意一个。
如果文档EF.CardAcess不可用，或者不包括PACE参数，查验系统应该利用BAC读取eMRTD

### 2 PACE
建议该步骤在eMRTD芯片支持PACE的情况下使用。
- 查验系统根据MRZ分散密钥$K_\pi$，如果查验系统知道CAN（Card Acess Number），则可以使用CAN代替MRZ。
- eMRTD应将MRZ作为PACE的口令，也可以用CAN作为口令。
- 查验系统和eMRTD芯片使用$K_\pi$互相认证，并分散出会话密钥$KS_{ENC}$和$KS_{MAC}$，应使用PACE协议。
如果成功，eMRTD应当执行以下操作：
- 启动安全通信
- 允许访问不敏感数据(例如DG1,DG2,DG14,DG15等，$SO_D$（Document Security Object）定义在敏感数据中)
- 限制访问权限以要求安全通信

### 3 选择eMRTD应用

### 4 BAC

如果eMRTD实施芯片访问控制，但是没有使用PACE，那么必须要有这一步，如果芯片成功实施了PACE，这一步就可以跳过。
- 查验系统从MRZ中分散基本访问密钥$K_{ENC}和K_{MAC}$
- 查验系统和eMRTD芯片使用证件基本访问密钥相互认证，并传出会话密钥$KS_{ENC}和KS_{MAC}$
如果成功，eMRTD应当执行以下操作：
- 启动安全通信
- 允许访问不敏感数据(例如DG1,DG2,DG14,DG15等，$SO_D$（Document Security Object）定义在敏感数据中)
- 限制访问权限以要求安全通信
 **查验系统必须使用DG14验证文件EF.CardAcess内容的真实性**

#### 4.3 BAC协议

##### 4.3.1 协议规范

Authentication和Key Establishment是根据**ISO/IEC 11770-2**的Key Establishment机制6，使用3DES，通过三轮挑战应答协议实现的。A cryptographic checksum是根据**ISO/IEC 9797-1**的MAC算法3计算，并附加到加密数据上。必须使用4.3.3节中描述的运算方式。被交换的随机数必须为8个字节，被交换的密钥材料为16个字节。接口设备IFD（查验系统）和非接触式IC不得使用特异标识符作为随机数。
具体地讲，IFD和IC应实施下列步骤：
- 1) IFD发送GET CHALLENGE命令，请求RND.IC，IC生成并返回随机数RND.IC
- 2) IFD执行下列操作：
    - a) 生成一个随机数RND.IFD和密钥材料K.IFD
    - b) 生成S=RND.IFD||RND.IC||K.IFD
    - c) 计算密文$E_{IFD}=E(K_{ENC},S)$
    - d) 计算MAC $M_{IFD}=MAC(K_{MAC},E_{IFD})$
    - e) 发送EXTERNAL AUTHENTICATE命令，使用数据$E_{IFD}||M_{IFD}$实现外部认证功能
- 3) IC执行下列操作：
    - a) 检验密文$E_{IFD}$的MAC $M_{IFD}$
    - b) 解密密文$E_{IFD}$
    - c) 从S中提取RND.IC,并检查IFD是否返回了正确的随机数
    - d) 生成密钥材料K.IC
    - e) 生成 R= RND.IC||RND.IFD||K.IC
    - f) 计算密文$E_{IC}=E(K_{ENC},R)$
    - g) 计算MAC $M_{IC}=MAC(K_{MAC},E_{IC})$
    - h) 返回响应数据$E_{IC}||M_{IC}$
- 4) IFD执行下列操作：
    - a) 检查密文$E_{IC}$的MAC $M_{IC}$
    - b) 解密密文$E_{IC}$
    - c) 从R中提取RND.IFD,并检查IC是否返回了正确的随机数
- 5) IFD和IC以K.IC和K.IFD作为密钥种子，使用9.7.4的密钥分散机制产生会话密钥$KS_{ENC}和KS_{MAC}$。

##### 4.3.2 查验过程

当实施BAC的eMRTD提供给查验系统查验时，使用光学或者视读的方式读取的信息分散得到证件的基本访问密钥($K_{ENC}和K_{MAC}$)，以便访问非接触IC，并且建立查验系统和非接触IC之间的安全通信信道。
建立安全信道后，支持BAC的eMRTD对任何未使用安全通信的APDU，做出没有**满足安全状态（SW:6982）**的响应。如果IC在安全信道建立之前或者安全信道终端之后发出普通指令SELECT，IC返回SW:6982或者9000，两种响应都符合ICAO的规定。
为了认证查验系统，必须实施以下步骤：
- 1) 查验系统读取**MRZ_information**,**MRZ_information**由证件号码、出生日期和到期日期拼接构成，并包括他们各自的校验位（ICAO Doc9303第4部分、第5部分或者第6部分分别对TD3、TD1和TD2这三种尺寸的证件的校验位做了描述）。**MRZ_information**通过使用OCR-B阅读器从MRZ读取，或者通过手工输入。这种情况下，手工输入的信息必须与MRZ中显示的信息保持一致。**MRZ_information**的SHA-1散列值中最重要的16个字节比用作密钥种子，通过第9.7.2节描述的密钥分散机制产生证件基本范文密钥。
- 2) 查验系统和eMRTD的非接触IC相互认证产生会话密钥，必须使用上文描述的认证和密钥建立协议。
- 3) 在成功执行认证协议后IFD和IC以（K.IC xor K.IFD）作为密钥种子，通过使用9.7.40描述的机制产生会话密钥$KS_{ENC}和KS_{MAC}$。随后的通信必须通过第9.8节中描述的安全通信加以保护。

##### 4.3.4 密码规范

###### 4.3.4.1 Challenge和Response的加密

使用**ISO/IEC 11568-2**中的0初始化向量zero IV(0x00 00 00 00 00 00 00 00)的双密钥3DES CBC模式来计算$E_{IFD}和E_{IC}$。在执行EXTERNAL AUTHENTICATE命令时，禁止对输入数据进行填充。

###### 4.3.4.2 Challenge和Response的认证

使用**ISO/IEC 9797-1**中的MAC算法3计算MAC $MAC_{IFD}和M_{IC}$,其中用到了分组密码DES、0初始化向量zero IV（8字节）和[**ISO/IEC 9797-1**中的填充方法2](https://www.jianshu.com/p/869cc3c5068b).MAC长度必须为8字节。

##### 4.3.4 APDU

APDU使用**ISO/IEC 7816-4**中的规定编码

###### 4.3.4.1 取随机数 Get Challenge

|C-APDU|
|---|
|CLA|||
|INS|'84'|Get Challenge|
|P1|'00'||
|P2|'00'||
|DATA|||

|R-APDU|
|---|
|DATA|随机数|
|SW|'9000'|成功|

###### 4.3.4.2 外部认证 External Authenticate

|C-APDU|
|---|
|CLA|||
|INS|'82'|External Authenticate|
|P1|'00'||
|P2|'00'||
|DATA||$E_{IFD}\|\|M_{IFD}$|

|R-APDU|
|---|
|DATA|随机数|
|SW|'9000'|成功|

#### 4.4 PACE

PACE是口令认证的Diffie-Hellman密钥协商协议，提供eMRTD芯片和查验系统的安全通信和基于口令的认证（即eMRTD芯片和查验系统共享同一个口令$\pi$）.
PACE在弱口令的基础上确立eMRTD芯片和查验系统之间的俺去那通信，在主文件中确立安全环境。协议使eMRTD芯片能够核验查验系统访问数据存储权限并具有下列特征：
- 不受口令强度影响，提供强会话密钥
- 用来认证查验系统的口令熵可能非常低（6位数足矣）
PACE使用的密钥$K_{\pi}$是由密钥分散函数$KDF_{\pi}$( 参见Doc 9303 part 11 9.7.3)从口令分散得到。对于有全球操作性的eMRTD，可以使用下列两种口令和相应的密钥：
- MRZ：由$K_{\pi}=KDF_{\pi}(MRZ)$定义的密钥$K_{\pi}$是必要的。类似于BAC，该密钥是从机读区（MRZ）分散而来，即由证件号码、生日、到期日分散而来。
- CAN：由$K_{\pi}=KDF_{\pi}(MRZ)$定义的密钥$K_{\pi}$是可选的。该密钥是由CAN分散而来，CAN是因在数据页正面的数字，必须随机或者伪随机选择（比如使用加密型强PRF）。
*注意： 与MRZ(证件号、出生日期、到期日)相反，CAN的优点是便于人工键入。*

作为协议执行的一部分，PACE支持不同的映射：
- 基于Diffie-Hellman密钥协商的通用映射；
- 基于域元素到密码群的嵌入映射的合成映射；
- 芯片认证映射扩展通用映射，并将芯片认证纳入PACE协议
如果芯片支持芯片认证映射，则芯片也必须支持通用映射或合成映射中的一种以及芯片认证。这意味着对于支持PACE的查验系统来说，仅支持通用映射和集成映射是必要的。支持芯片认证映射是可选的。

##### 4.4.1 协议规范

查验系统从文件EF.CardAcess读取eMRTD芯片支持的PACE参数，选取要使用的参数，随后执行协议。
应使用下列命令：
- Doc 9303第10部分规定的READ BINARY
- 4.4.4.1中规定的MSE:SET AT(具有设定认证模板功能的MANAGE SECURITY ENVIRONMENT命令)
- 查验系统和eMRTD芯片使用第4.4.4.2中规定的一系列通用认证命令实施下列步骤：
    - 1) eMRTD芯片选择随机数s，将该随机数加密为$z= E(K_{\pi},s)$，其中$K_{\pi}=KDF_{\pi}({\pi})$是从共享口令$\pi$中分散的来，并将密文$z$发送给查验系统
    - 2) 查验系统在共享口令$\pi$的帮助下，恢复出明文$s=D(K_{\pi},z)$
    - 3) eMRTD芯片和查验系统两者都执行下列步骤：
        - a) 交换随机数映射必须的其他数据：
            - i) 对于通用映射，eMRTD芯片和查验系统交换临时密钥公钥
            - ii) 对于合成映射， 查验系统向eMRTD芯片发送另一个随机数
        - b) 按照4.4.3.3的描述，计算临时域参数$D=Map(D_{IC},s,\cdots)$
        - c) 在临时域参数的基础上进行匿名Diffie_Hellman密钥协商，生成共享密钥$K=KA(SK_{DK,IC},PK_{DH,PCD},D)=KA(SK_{DC,PCD},PK_{DH,IC},D)$
        - d) 在Diffie-Hellman密钥协商的过程中，继承电路和查验系统应验证两个公钥$PK_{DH,IC}和PK_{DH,PCD}$是不同的
        - e) 按照Doc 9303 Part.11 9.7.1的描述，产生会话密钥$KS_{MAC}=KDF_{MAC}(K)和KS_{ENC}=KDF_{ENC}(K)$
        - f) 按照4.4.3.4的描述，交换并核验认证令牌(authentication token)$T_{PCD}=MAC(KS_{MAC,PK_{DH,IC}})和T_{IC}=MAC(KS_{MAC},PK_{DH,PCD})$
    - 4) 在一定条件下，eMRTD芯片计算芯片认证数据$CA_{IC}$，将数据加密$A_{IC}=E(KS_{ENC},CA_{IC})$，并将加密数据发送给终端，终端解密$A_{IC}$，通过使用经恢复的芯片认证数据$CA_{IC}$来证实芯片的真实性。

##### 4.4.2 安全状态

支持PACE的eMRTD芯片对应非授权的阅读企图（包括对逻辑数据结构中的被保护文件的选择）做出**“没有满足安全状态”（SW:6982）**的响应。
*注意：该规范比仅支持BAC的eMRTD的相应规范更具限制性。*
如果成功进行了PACE，eMRTD便核验了所使用的口令。使用分散得到的会话密钥$KS_{MAC}和KS_{ENC}$启动安全通信。

##### 4.4.3 密码规范

eMRTD的签发者选择特定算法。查验系统必须支持下列所有算法的组合，eMRTD芯片可支持一种以上算法组合。
*注意：一些算法不可用于芯片认证映射：鉴于安全原因，不再建议使用3DES。DH-variants are not available to reduce the number of variants to be implemented by Terminals.。*

###### 4.4.3.1 DH

关于DH的PACE，必须使用Doc 9303 part.11 9.6中和下表的响应算法和格式。

**DH的算法和格式**
|OID|Mapping|Sym.Cipher|Key-length|Secure Messaging|Auth.Token|
|---|---|---|---|---|---|
|id-PACE-DH-GM-3DES-CBC-CBC|Generic|3DES|112|CBC / CBC|CBC|
|id-PACE-DH-GM-AES-CBC-CMAC-128|Generic|AES|128|CBC / CMAC|CMAC|
|id-PACE-DH-GM-AES-CBC-CMAC-192|Generic|AES|192|CBC / CMAC|CMAC|
|id-PACE-DH-GM-AES-CBC-CMAC-256|Generic|AES|256|CBC / CMAC|CMAC|
|id-PACE-DH-IM-3DES-CBC-CBC|Integrated|3DES|112|CBC / CBC|CBC|
|id-PACE-DH-IM-AES-CBC-CMAC-128|Integrated|AES|128|CBC / CMAC|CMAC|
|id-PACE-DH-IM-AES-CBC-CMAC-192|Integrated|AES|192|CBC / CMAC|CMAC|
|id-PACE-DH-IM-AES-CBC-CMAC-256|Integrated|AES|256|CBC / CMAC|CMAC|

###### 4.4.3.2 ECDH

关于ECDH下的PACE，必须使用Doc 9303 Part.11 9.6和下表的算法和格式。
应仅使用带有未压缩点的素数曲线。应该使用Doc 9303 Part.11 9。5.1中描述的标准化域参数。
**ECDH的算法和格式**
|OID|Mapping|Sym.Cipher|Key-length|Secure Messaging|Auth.Token|
|---|---|---|---|---|---|
|id-PACE-ECDH-GM-3DES-CBC-CBC|Generic|3DES|112|CBC / CBC|CBC|
|id-PACE-ECDH-GM-AES-CBC-CMAC-128|Generic|AES|128|CBC / CMAC|CMAC|
|id-PACE-ECDH-GM-AES-CBC-CMAC-192|Generic|AES|192|CBC / CMAC|CMAC|
|id-PACE-ECDH-GM-AES-CBC-CMAC-256|Generic|AES|256|CBC / CMAC|CMAC|
|id-PACE-ECDH-IM-3DES-CBC-CBC|Integrated|3DES|112|CBC / CBC|CBC|
|id-PACE-ECDH-IM-AES-CBC-CMAC-128|Integrated|AES|128|CBC / CMAC|CMAC
|id-PACE-ECDH-IM-AES-CBC-CMAC-192|Integrated|AES|192|CBC / CMAC|CMAC|
|id-PACE-ECDH-IM-AES-CBC-CMAC-256|Integrated|AES|256|CBC / CMAC|CMAC|
|id-PACE-ECDH-CAM-AES-CBC-CMAC-128|Chip Authentication|AES|128|CBC / CMAC|CMAC|
|id-PACE-ECDH-CAM-AES-CBC-CMAC-192|同上|AES|192|CBC / CMAC|CMAC|
|id-PACE-ECDH-CAM-AES-CBC-CMAC-256|同上|AES|256|CBC / CMAC|CMAC|

###### 4.4.3.3 加密和映射随机数 Encrypting and Mapping Nonces

eMRTD芯片应选择随机数s，该随机数是长度为l的二进制比特串，l是eMRTD新派你选择的相应分组密码$E()$分组长度的倍数。
-  随机数（nonce）s应使用从口令$\pi$分散的密钥$K_{\pi}=KDF_{\pi}(\pi)$和设为全0的初始化向量，按照**ISO/IEC 10116**以CBC模式进行加密。
- 使用特定映射函数Map，将随机数（nonce）s转换为随机数发生器（random generator）
- 对应合成映射（ingegrated Mapping），应该额外再选取一个随机数（nonce）t作为长度为k的二进制比特串，并以明文发送。在这种情况下，k是相应分组密码$E()$的密钥长度，l是使得$l\ge k$的分组密码$E()$分组长度的最小倍数。
将随机数s或者随机数s，t映射入密码群，应采用下列映射的一种：
- 通用映射
- 合成映射
- 芯片认证映射

###### 4.4.3.3.1 通用映射 Generic Mapping

**ECDH**
函数$Map:G\to \hat{G}$被定义为$\hat{G}=s\times G+H$，其中选择$<G>$中的H，使得$log_{G}H$未知。H点应由匿名Diffie-Hellman密钥协商进行计算，即$H=KA(SK_{Map,IC},PK_{Map,PCD},D_{IC}=KA(SK_{Map,PCD},D_{IC})$。
*注意：密钥协商算法ECKA通过使用兼容余因子乘法来避免小子群攻击。*

**DH**

函数$Map:g\to \hat{g}$被定义为$\hat{g}=g^s \times h$，其中，选择<g>中的h，使得$log_g h$未知。群元素h应按照$h=KA(SK_{Map,IC},PK_{Map,PCD},D_{IC})=KA(SK_{Map,PCD},PK_{Map,IC},D_{IC})$。
*注意：必须使用**RFC2631**描述的公钥验证方法来避免小子群攻击。 *

###### 4.4.3.3.2 合成映射 Integrated Mapping

###### 4.4.3.3.3 芯片认证映射 Chip Authentication Mapping

##### 4.4.3.4 认证令牌 AUthencation Token

