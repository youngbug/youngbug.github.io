---
layout: post
title: CCC 数字钥匙Release 3 学习笔记 车主配对命令 Part2
time: 2022年9月9日
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: cryptography
tags: [cryptography]
excerpt_separator:  <!--more-->
---
整理了一下CCC组织的汽车数字钥匙Release 3中关于车主配对Owner Paring，过程的APDU指令和数据说明。基本可以算是在车端的角度进行车主配对操作。里面的章节表格编号，都按照CCC数字钥匙Release 3文档中的编号走，方便将来检索对照。

## 车主配对命令

<!--more-->

### 5.1 车主配对的指令

*表5-1：车主配对命令集*

|命令|Ins Byte(HEX)|实现方|
|-|-|-|
|SELECT|A4|Digital Key Framework|
|SPAKE2+REQUEST|30|Digital Key Framework|
|SPAKE2+VERIFY|32|Digital Key Framework|
|WRITE DATA|D4|Digital Key Framework|
|GET DATA|CA|Digital Key Framework|
|GET RESPONSE|C0|Digital Key Framework|
|OP CONTROL FLOW|3C|Digital Key Framework|
|参见表15-1|5.3.2|Digital Key Applet|

*表5-2 基本状态字*

|SW1SW2|描述|
|-|-|
|6700|长度错|
|6A80|数据域的参数错|
|6A82|文件找不到|
|6B00|P1 P2错|
|6C00|Le错|
|6D00|INS错|
|6E00|CLA错|
|9000|成功|

### 5.1.1 SELECT指令

汽车向钥匙设备发送SELECT AID指令。Digital Key framework AID为 *A000000809434343444B467631*。

当Digital Key framework被选中，设备应当按照表5-3返回数据。

钥匙设备应当向车辆指示当前配对状态，可能状态有：

- 未配对
- 配对模式开始且配对口令已经输入

SELECT指令用来选择Digital Key applet实例（使用实例AID）在15.3.2.1定义。

C-APDU: 00 A4 04 00 Lc [Digital_Key_Framework_AID] 00

R-APDU: [表 5-3]90 00 

*表 5-3 SELECT指令响应*

|Tag|长度(bytes)|描述|是否必须|
|-|-|-|-|
|5A|2*n|n支持SPAKE2+协议版本（ver.high\|ver.low）| 必须 |
|5C|2*m|m支持的Digital Key applet协议版本（ver.high\|ver.low）|必须|
|D4|1|00 = 未配对<br/>02 = 配对模式开始且配对口令已经输入|必须|

钥匙设备应当返回所有支持的的SPAKE2+和支持的Digital Key applet协议版本，每个版本使用2字节大端编码的数据。

在CCC Release 3的规范里，钥匙设备应当支持SPAKE2+协议版本1.0（编码0100）和Digital Key applet协议1.0（编码0100）。车辆应当依据6.3.3.8来匹配支持的版本。

钥匙设备是否处于配对模式时会发出信号，如果钥匙设备是“未配对”模式，车辆只有在自己处于已配对模式时才能继续（行驶）。

### 5.2.1 SPAKE2+REQUEST指令

在这个指令中，车辆应当发送所选择的SPAKE2+版本，所有支持的Digital Key applet协议版本和SPAKE2+的Scrypt参数。车辆应当在响应中获取SPAKE2+协议的曲线点X。SPAKE2+REQUEST指令使用的参数见18节。

C-APDU：80 30 00 00 Lc [表 5-4] 00 

R-APDU：[表 5-5] 90 00

*表5-4 SPAKE2+REQUEST指令*

|Tag|长度(bytes)|描述|是否强制|
|-|-|-|-|
|5B|2|同意的SPAKE2+协议版本|强制|
|5C|2*m|m支持的Digital Key applet协议版本（ver.high\|ver.low）|强制|
|7F50|32|Scrypt配置参数|强制|
| - C0|16|密码盐值 s|强制|
| - C1|4|Scrypt cost参数， $N_{scrypt}$|强制|
| - C2|2|块大小参数 r|强制|
| - C3|2|平行化参数 p|强制|
|D6|2|车辆品牌（见表2-1）包含这个Tag，包括只支持NFC的的车辆|强制|

*表5-5 SPAKE2+REQUEST 响应*

|Tag|长度(bytes)|描述|是否强制|
|-|-|-|-|
|50|65|SPAKE2+协议的曲线点X，prepended with 04 h as per Listing 18-3|强制|

在发送SPAKE2+REQUEST指令前，车辆应当先检查SPAKE2+配对计数器。如果计数器指示已经配对7次，车辆应当不再发送配对指令。相反的，车辆应当发送一个OP CONTROL FLOW来中止（超过错误计数器次数、需要新的配对口令或者没有明确原因）。当新的配对口令产生，计数器将被复位。

车辆应当发送所选的2字节的SPAKE2+协议版本。

车辆也应当发送支持的Digital Key applet 协议版本列表，借此第一个被列出的版本应当被车主设备使用。整个Digital Key applet协议版本列表（Tag 5C）应当被包含在Key Creation Request（见11.8.1，DIGITAL_KEY_APPLET_PROTOCOL_VERSION）。这个允许在分享钥匙到好友设备时选择最好的版本使用。

SPAKE2+协议所有的曲线点的定义遵守X9.63标准，格式为 0x04\|\|\<x\>\|\|\<y\>的字节流，x和y为32字节的大端表示（见18.1）。

如果返回的X值在无穷远或者不是一个在椭圆曲线上定义的合法的点，车辆应当中止流程，并发送OP CONTROL FLOW指令，按照5.1.7的描述P2值设置为0C。

Scrypt的迭代次数（cost参数）是一个4字节的无符号整数，用来配置Scrypt功能（见18.4）在主机厂服务器和钥匙设备上来派生校验者的值。其他传输的Scrypt参数还有块大小和平行化参数（见18.1.2）。Scrypt cost参数、块大小参数、平行化参数的TLV值部分都是编码为大端格式。

如果车辆没有找到双方支持的SPAKE2+或者Digital Key applet协议版本，车辆应当发送一个OP_FLOW_CONTROL(Owner Paring
 FLow Control)指令包含表5-24中定义的原因代码，代替SPAKE2+REQUEST指令。

如果SPAKE2+REQUEST指令被成功处理，车辆和钥匙设备应当计算共享密钥K，分别为Listing 18-4和Listing 18-5。

*表5-6 SPAKE2+REQUEST响应错误状态字*

|SW1SW2|描述|
|-|-|
|6985|指令使用顺序不对|
|6A88|收到的数据不合法或者为0|
|9484|设备配对还未准备就绪|

### 5.1.3 SPAKE2+VERIFY 指令

这个指令相互交换证据来证明双方计算出来的共享密钥是相同的。

C-APDU: 80 32 00 00 Lc [表 5-7] 00

R-APDU: [表 5-8] 90 00 

*表5-7 SPAKE2+VERIFY指令*

|Tag|长度(bytes)|描述|是否强制|
|-|-|-|-|
|52|65|SPAKE2+协议的曲线点Y，prepended with 04 h as per Listing 18-2|强制|
|57|16|车辆证据M[1]|强制|

*表5-8 SPAKE2+VERIFY响应*

|Tag|长度(bytes)|描述|是否强制|
|-|-|-|-|
|58|16|钥匙设备证据 M[2]|强制|

在发送SPAKE2+VERIFY指令之前，汽车应当先给SPAKE2+配对尝试计数器加1。

汽车应当计算证据M[1]（描述在Listing 18-7）并发送它给钥匙设备，一起发送的还有曲线点Y。

钥匙设备应当验证下面的内容：

- 1. 收到的曲线点Y是否是定义在椭圆曲线上的合法的点
- 2. 收到M[1]

如果都验证成功，钥匙设备应当计算证据M[2]（描述Listing 18-8），并在SPAKE2+响应中将M[2]给车辆返回。

只有车辆成功验证了收到的M[2]，车辆才能继续车主配对流程。

如果上面任何验证失败，比如钥匙不能计算M[2]且不能返回M[2]或者返回其他除了状态字之外的响应。这种情况车辆应当发送一个OP_CONTROL_FLOW指令按照5.1.7中的描述将P2设置未09去中止车主配对.

*表5-9 SPAKE2+VERIFY响应状态字*

|SW1SW2|描述|
|-|-|
|6985|指令使用顺序不对|
|6A88|验证证据失败|

SPAKE2+VERIFY步骤引出用于建立钥匙框架和车辆交换后续指令的SCP03通道的安全通道密钥。建立SCP03通道的遵守Listing 18-10和Listing 18-11。创建安全通道的密钥在下列情况应当被复位：

- 在成功配对之后
- 当钥匙设备响应的状态字不是9000或者6100时
- SPAKE2+REQUEST指令已经发送
- 当出现通信中断时
- 车主配对没有终止，但时间超过最大允许的时间

车辆需要再重新开始SPAKE2+建立新密钥。注意这种情况配对口令还是原来的。车辆应当在7次尝试失败后更换口令。

### 5.1.4 WRITE DATA指令

这个指令向钥匙设备发送生成数字钥匙所需要的所有数据。它也用于提供证明车辆的密钥追踪用途的设备公钥。(好蹩脚，凑合看)

C-APDU: 84 D4 P1 00 Lc [command_data] [command_mac] 00
R-APDU: [response_mac] 90 00 

除了发最后一次WRITE DATA指令时P1应当设置为80外，其他情况下参数P1和P2永远设置为0。

这个指令只允许在安全通道内发送。

*表5-10 WRITE DATA响应状态字*

|SW1SW2|描述|
|-|-|
|6985|指令使用顺序不对|
|6A84|内存不够|

一个或者多个WRITE DATA指令可以用来向钥匙设备写入请求的数据对象。

*表5-11 数字钥匙创建数据对象*

|Tag|长度(bytes)|数据内容|是否强制|
|-|-|-|-|
|7F4A|var|端点创建数据，元素来自表15-13|强制|
|7F4B|var|车辆公钥证书K，Der编码X509证书，Listing 15-3|强制|
|7F4C|var|中间证书， Der编码的X509证书|可选|
|7F4D|var|邮箱映射 表5-13|强制|
|7F4E|var|设备配置 表5-14|强制|
|5F5F|0|数字钥匙数据发送完成|强制|

*表5-12 设备数字钥匙证书的对象*

|Tag|长度(bytes)|数据内容|是否强制|
|-|-|-|-|
|5F5A|var|车辆对设备的认证公钥（不透明）|可选|
|5F5F|0|数字钥匙证书发送完成|强制|

表5-11中所有要求的的对象应当按照表格顺序被写入设备。

一个或者多个TLV允许被写在每个WRITE DATA指令中。

TLV 5F5F应当在最后被写入标记数据写入结束。最后一个WRITE DATA指令应当通过设置P1=80指示包含TLV 5F5F。

当车辆已经接受了设备公钥，表5-12中所有要求的对象应当被写入钥匙设备。最后一个WRITE DATA指令应当通过设置P1=80来指示。

5F5F 应当为最后一个被写入的TLV，标记传输数据结束。

Tag 7F4A应当包含表15-13中所有数据元素，除了下面的元素：

- 端点标识符（设备定义）
- Instance CA标识符 （设备定义）
- 计数器限制（弃用，不使用）

如果车辆标识符作为7F4A的一部分提供，设备应当比较它的值和车辆公钥叶子证书K的值，如果检查失败车主配对应当中止。

最大指令数据长度应道为239字节：len=[command_data] + [padding] + [command_mac]

len = 239 + 1 + 8 <= 255(ok)

len = 240 + 16 + 8  255(not ok)

[command_data]+[padding]应道是AES分组16字节的整数倍。填充方案描述在9.至少填充1字节的80。最大响应数据长度应当为239字节。

车辆公钥应当以被主机厂CA签发的一个X509证书形式提供，描述在Listing 5-3。

*Listing 5-1 车辆证书扩展方案*

```
VehicleCertificateExtensionSchema ::= SEQUENCE
{
 extension_version INTEGER (1..255)
}
```

*Listing 5-2 车辆证书扩展数据*

```
vehicle-cert-extension-data VehicleCertificateExtensionSchema ::=
{
 extension_version 1 --value shall be 1
}
```

车辆公钥证书数据描述在Listing 5-3。

*Listing 5-3 车辆公钥证书数据*

```
vehicle-key-cert-data Certificate ::=
 {
 tbsCertificate
 {
 version v3, --shall be v3--
 serialNumber ..., --a random integer chosen by the certificate issuer,
 Signature
 {
 algorithm {1 2 840 10045 4 3 2} --OID for ecdsaWithSHA256 (ANSI X9.62 ECDSA algorithm with SHA-256)
 },
 issuer rdnSequence:
 {
 {
 {
 type {2 5 4 3}, --OID for CommonName
 value "..." --shall match the subject of the issuing certificate, shall use PrintableString or UTF8String format

 }
 }
 },
 validity
 {
 notBefore Time: "..." --shall use UTCTime or GeneralizedTime as defined in [3]
 notAfter Time: "..." --shall use UTCTime or GeneralizedTime as defined in [3]
 },
 subject rdnSequence:
 {
 {
 {
 type {2 5 4 3}, --OID for CommonName
 value "Vehicle OEM Identifier" --contains the subject of the certificate, as per Appendix B.2.6
 shall use PrintableString or UTF8String format
 }
 }
 },
 subjectPublicKeyInfo
 {
 algorithm
 {
 algorithm {1 2 840 10045 2 1} --OID for ecPublicKey (ANSI X9.62 public key type)
 parameters {1 2 840 10045 3 1 7} --OID for prime256v1(ANSI X9.62 named elliptic curve)
 },
 subjectPublicKey '04...'H --the public key pre-pended with 04 h to indicate uncompressed format
 },
 extensions
{
{
extnID {1.3.6.1.4.1.41577.5.1}, --OID for Vehicle Public Key Certificate (see Appendix B.2.2)
critical TRUE,
extnValue ‘…’H --DER encoding for VehicleCertificateExtensionSchema extension as per Listing 5-1
},
 {
 extnID {2 5 29 15}, --KeyUsage standard extension
 critical TRUE,
 extnValue '03020780'H --DER encoding for KeyUsage, digitalSignature only
 },
 {
 extnID {2 5 29 19}, --BasicConstraints standard extension
 critical TRUE,
 extnValue '3000'H -- DER encoding for cA=FALSE
 },
 {
 extnID {2 5 29 35}, --OID for AuthorityKeyIdentifier standard extension
 critical FALSE,
 extnValue '...'H- DER encoding of an AuthorityKeyIdentifier sequence, containing only a KeyIdentifier element.
 -- The KeyIdentifier is an OCTET STRING containing the 160-bit SHA-1 hash of the value of the BIT STRING
subjectPublicKey
 --from the issuer certificate (excluding the tag, length, and number of unused bits)
 },
 {
 extnID {2 5 29 14}, -- OID for SubjectKeyIdentifier standard extension
 critical FALSE,
 extnValue ‘…’H --160-bit SHA1 hash of the value of the BIT STRING subjectPublicKey
--(excluding the tag, length, and number of unused bits)
}
 }
 },
 signatureAlgorithm
 {
 algorithm {1 2 840 10045 4 3 2}
 },
 signatureValue '...'H --the certificate signature computed as per [3]
 --ECDSA signature
 }
```

### 5.1.5 GET DATA指令

这个指令应当继续使用已经建立的会话密钥去检索所有需要的所有数据，验证Digital Key framework创建在Digital Key applet 实例中的数字钥匙。

C-APDU: 84 CA 00 00 Lc [encrypted_tag] [command_mac] 00

R-APDU: [response_payload] [response_mac] 90 00 or 61XX

每个GET DATA指令一次只能请求一个Tag 。---->这个地方跟EMV或者其他应用一样。

X509证书不应当在封装成TLV结构

### 5.1.6 GET RESPONSE指令

跟ISO 14443规定的一个样，这里就不写了。