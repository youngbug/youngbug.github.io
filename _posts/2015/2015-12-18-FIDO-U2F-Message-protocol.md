---
layout: post
title: FIDO U2F设备的Message协议
time: 2015年12月18日 星期五
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: Cryptography
tags: [cryptography, smartcard, u2f, fido]
excerpt_separator:  <!--more-->
---

这次说一下U2F应用层的协议，在FIDO的规范中叫做**U2F Raw Message Formats**。这个协议也就是在前两篇[FIDO U2F HID协议](https://youngbug.github.io/smartcard/2015/12/15/FIDO-U2F-HID-protocol.html)和[FIDO U2F NFC协议](https://youngbug.github.io/cryptography/2015/12/16/FIDO-U2F-NFC-protocol.html)之上的实现U2F应用的协议，其实就是U2F应用的智能卡APDU，这个应用真正实现U2F两步认证的应用注册和认证登录。

U2F的使用场景一般是分为注册和认证两个情形，下面简单的说一下U2F的大概使用过程。

<!--more-->

**注册**：

在支持U2F两步认证的系统中，比如Github等，首先在自己的账户中绑定U2F设备，先将自己的U2F令牌插入计算机，Github会通过浏览器向U2F令牌发送注册请求，这个注册请求包括了随机数、应用名称等要素，令牌给这个应用对应的生成一个用户密钥对和对应的key handle，并使用内部attestation certificate对应的私钥对用户公钥、应用参数和随机参数等信息进行签名，并将用户公钥、key handle，attestation certificate，前述签名等数据返回给应用。应用首先验证attestation certificate，并使用attestation certificate验证了签名，然后讲key handle和对应的用户公钥保存在用户账户中，完成U2F设备的绑定。

**认证**：

用户登录到支持U2F两步认证的系统时，用户在应用登录界面输入完账号和口令后，应用通过FIDO客户端向U2F令牌发送认证请求，认证请求包括随机参数，应用参数，key length，key handle。U2F令牌收到认证请求后，首先通过key length和key handle找到对应的注册信息，检查请求认证的应用是否就是最初注册的应用，如果检查通过，那么使用对应的用户私钥对应用参数、挑战信息、认证计数器等信息进行签名，然后讲签名、认证计数器返回给应用。应用使用注册时获得的用户公钥对签名进行验证，如果能验证通过则完成两步验证，应用登录成功，否则两步验证失败，拒绝登录。


## 1.U2F 消息封包

U2F协议是基于请求-响应方案的，当请求者发送一个请求消息到U2F设备中时，U2F会返回一个响应消息给请求者。

请求报文会被封装后发送到下一层。拿签名请求举例来说，U2F协议是FIDO客户端告诉下一层的通信层它要发送一个签名请求，然后发送原始消息内容。U2F协议也规定了当如何返回响应的原始消息，如果命令执行失败后的错误码。

在现在这一版的U2F协议中，消息的封包是基于ISO7816-4:2005 扩展APDU格式。

### 1.1 请求消息格式 Request Message Framing

原始的请求消息封装成一个APDU命令，APDU的格式如下：

**CLA INS P1 P2 \[Lc &lt;request-data&gt;\] \[Le\]**

- **CLA** : 保留给底层传输协议使用，比如ISO/IEC 7816里规定可以指示是否使用密文，是否计算MAC等等。这里应用应该设置CLA为0。

- **INS** : U2F 指令，具体的定义下面会写。

- **P1,P2** : 参数1和2，每个指令都有各自的定义。

- **Lc** : **request-data**的长度，如果没有request-data,Lc省略。

- **Le** : 期待返回的响应数据的长度，如果不存在响应数据，Le省略。

APDU的准确格式取决于编码的选择。允许使用两种不同的APDU编码：**short**和**extended length**。不同的地方在于请求数据的长度，Lc和期待响应数据的最大长度，Le的编码。

#### 1.1.1 指令和参数

|指令|INS|P1|P2|
|-|-|-|-|
|U2F_REGISTER|0x01|0x00|0x00|
|U2F_AUTHENTICATE|0x02|0x03\|0x07\|0x08|0x00|
|U2F_VERSION|0x03|0x00|0x00|
|VENDOR SPECIFIC|0x40-0xBF|NA|NA|

#### 1.1.2 短编码 Short Encoding

在**短编码**中，请求数据的最大长度是255字节。Lc按下面的方法编码：

Nc=\|\<**request-data**\>\|。如果Nc=0，那么Lc省略，否则使用Nc的值编码为一个字节赋值给Lc。

如果指令不期望应用返回任何响应数据，那么Le可以省略。否则在**短编码**中，Le按下面的方法编码：

Ne = 响应数据的最大值，在**短编码**中Ne的最大值为256字节。

Ne的值在1和255之间时，Le直接使用Ne的值。当Ne=256时，Le的值赋值为0。

#### 1.1.3 扩展长度编码 Extended Length Encoding

在**扩展长度编码**中，请求数据的最大长度为65535字节。Lc按下面的方法编码：

Nc= \|\<**request-data**\>\|。如果Nc=0，Lc省略。否则Lc这样编码：**0 MSB(Nc) LSB(Nc)**

这里**MSB(Nc)**是Nc的高字节，**LSB(Nc)**是Nc的低字节。

换一种说法，请求数据是以3个字节开头，第一个字节是0，然后后面跟着两个字节的请求数据长度（大端字节序）。

如果指令不期望应用返回任何响应数据，Le可以省略。否则在**扩展长度编码**中，Le按下面的方法编码：

Ne=响应数的最大长度。在**扩展长度编码**中，Ne的最大值为65535字节。

当Ne的值在1和65535之间时，Le1=MSB(Ne)，Le2=LSB(Ne)，这里**MSB(Nc)**是Nc的高字节，**LSB(Nc)**是Nc的低字节。

当Ne=65536，Le1=0，Le2=0。

当Lc存在的时候，比如Nc\>0，Le编码为：**Le1 Le2**

当Lc不存在的时候，比如Nc=0，Le编码为：**0 Le1 Le2**

换句话说，当Lc不存在的时候，Le有一个字节值为0的前缀。

### 1.2 响应消息格式 Response Message Framing

原始响应消息的格式如下：

&lt;request-data&gt; SW1 SW2

**SW1**和**SW2**是两个字节的状态字，状态字的定义在下面，SW1是高字节，SW2是低字节。

### 1.3 状态代码 Status Code

以下ISO7816-4中定义的的状态字，在U2F中有特殊的含义：

- **SW_NO_ERROR(0x9000)** : 指令执行成功，没有错误。
- **SW_CONDITIONS_NOT_SATISFIED(0x6985)** : 因为需要进行用户在场测试导致请求被拒绝
- **SW_WRONG_DATA(0x6A80)** : 因为非法的key handle导致请求被拒绝
- **SW_WRONG_LENGTH (0x6700)** : 请求长度不正确
- **SW_CLA_NOT_SUPPORTED (0x6E00)** : CLA字节不支持
- **SW_INS_NOT_SUPPORTED (0x6D00)** : 指令INS字节不支持

U2F的供应商可以定义其他供应商自定义的代码，提供更多的错误信息。U2F FIDO客户端只处理上面列出的错误代码。

## 2.注册消息

### 2.1注册消息 U2F_REGISTER

![img](/assets/blog_image/2015/20151218001.png)

注册消息是需要使用U2F认证的网站在第一使用时，需要注册U2F设备的一个操作，注册请求消息有两部分，如上图所示：

- **challenge parameter**[32 bytes]， challenge parameter是对客户端数据(Client Data)进行SHA-256运算得到的哈希值。Client Data是一个由FIDO客户端准备的JSON数据结构。

- **application parameter**[32 bytes]，application parameter是请求注册的应用id的SHA-256结果。

### 2.1 注册消息的响应: 错误 需要Test-of-User-Presence

如果U2F令牌无法获得Test-of-User-Presence测试，那么令牌将返回这个消息。

### 2.3 注册消息的响应： 成功

如果注册成功，U2F令牌会创建一个新的密钥对。需要注意的是U2F令牌在返回注册成功的响应消息之前，需要用户确认（一般是带触点按键的，按下按键或者触点，没有按键的一般是拔插一下设备），否则令牌应该返回错误码test-of-user-presence-required。

![img](/assets/blog_image/2015/20151218002.png)

如果注册成功，返回的响应消息格式如上图所示。

- **reserved byte** [1字节] 保留字节永远是0x05

- **user public key** [65bytes] 椭圆曲线公钥，x,y表示的P-256 NIST椭圆曲线上的点（未压缩）

- **key handle length byte** [1byte]  key handle的长度，值为0-255之间的无符号数

- **key handle** [长度根据上面handle lenght确定] U2F令牌用来区分产生的密钥对的

- **attestation certificate** [变长] X.509 DER格式的证书；解析X.509证书可以明确的确定结尾，剩下的字节就是signature了

- **signature** [变长，71-73字节] ECDSA签名(在P-256上)对下面数据进行的签名，签名的数据为0x00,application parameter, challenge parameter,key handle,user public key

签名依赖方可以使用attestation certificate中的公钥进行验证。依赖方也应该验证attestation certificate是可信的CA颁发的。

一旦依赖方验证了签名，它就应该保存public key和key handle，这样就可以在之后的认证操作中使用了。

>**说明**  
>这里的attestation certificate是一批硬件里都用相同的证书，并不会每一个U2F硬件单独使用一个证书。曾经某个互联网大厂搞物联网安全的负责人说，怎么可能一堆硬件都用同样的证书呢，肯定是每个硬件单独有证书如何如何。实际上FIDO这里设计的是，一批硬件都用相同的attestation certificate，如果这批硬件有风险，直接吊销这批硬件使用的证书即可停用整批硬件。
>

## 3.认证消息

### 3.1 认证请求消息 U2F_AUTHENTICATE

![img](/assets/blog_image/2015/20151218003.png)

这个消息是U2F令牌进行认证的指令，FIDO客户端首先连接依赖方获得一个挑战值（随机数），然后构造认证请求报文下发到U2F令牌中，认证请求报文有下面5部分：

- control byte[P1] 

  - 值取0x07的时候表示check-only，U2F令牌只是简单的检查传入的key handle是不是这个令牌产生的，并且是不是给这个应用创建的。如果是，U2F令牌必须返回一个认证响应。

  - 值取0x03时，表示强制用户出示并签名，U2F令牌执行一个真实的签名并响应一个认证应答消息（好别扭）

  - 值取0x08的时候表示不强制用户出示并签名，在不用验证用户存在的情况下，就让U2F令牌执行签名

- challenge parameter [32bytes] Client Data的SHA-256哈希数据，Client Data是FIDO客户端生成的一个json数据结构

- application parameter [32bytes]  是请求注册的应用id的SHA-256结果

- key handle length byte [1 byte] key handle的长度，取值为0-255的无符号数

- key handle [变长] 注册时获得的key handle

### 3.2 注册指令的响应：错误 需要Test-of-User-Presence

### 3.3 注册指令的响应：错误 Bad KeyHandle

U2F令牌如果发现key handle不是自己创建的，或者key handle是这个U2F令牌创建的，但是是对应其他的application parameter，直接返回Bad Key Handle错误。

### 3.4 注册指令的响应：成功

如果认证成功，返回响应如下

![img](/assets/blog_image/2015/20151218004.png)

U2F令牌在处理完认证请求消息后，返回上面的消息。

- user presence byte [1 byte] bit0设置为1，表示用户确认过了（这一版的协议不明确的要求在进行认证请求是用户确认的方式）。这个字节剩下的bit，FIDO保留以后使用，现在都设置为0即可。

- counter [4bytes] 一个大端的计数器，U2F令牌每次认证都增加一次。

- signature 对以下数据的ECDSA签名（P-256）:

   - application parameter 来自认证请求
   - user presence byte [1 byte] 参看上面定义
   - counter [4 bytes] 参看上面的定义
   - challenge paraeter [32 bytes] 来自认证请求

签名使用[ANSI X9.62格式](http://webstore.ansi.org/RecordDetail.aspx?sku=ANSI+X9.62%3A2005)进行编码。

依赖方使用注册时获得的的公钥来验证签名。

## 4 其他消息 Other Messages

### 4.1 获得版本号 U2F_VERSION

FIDO客户端用来获得U2F协议的版本。版本的描述在U2F_V2这个文档中。

响应消息的原始数据为ASCII编码的字符串'U2F_V2',不含引号和任何结束符。

短编码APDU指令如下:

|CLA |INS |P1 |P2 |Le|
|-|-|-|-|-|
|00 |03 |00 |00 |00|

扩展长度编码APDU指令如下:

|CLA |INS |P1 |P2 |Le|
|-|-|-|-|-|
|00 |03 |00 |00 |00 00 00|

### 4.2 扩展和厂商定义指令

U2F_VENDOR_FIRST 和 U2F_VENDOR_LAST之间的指令可以用来作为厂商定义指令。比如厂商可以加一些测试用的指令。FIDO客户端永远不会生成调用这些命令的请求。这个范围之外的指令为RFU或者还没使用。
