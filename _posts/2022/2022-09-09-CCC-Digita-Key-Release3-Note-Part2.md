---
layout: post
title: CCC 数字钥匙Release 3学习笔记 Part2
time: 2022年9月9日
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: cryptography
tags: [cryptography]
excerpt_separator:  <!--more-->
---
## 车主配对命令



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

汽车向钥匙设备发送SELECT AID指令。Digital Key framework AID为**A000000809434343444B467631**。
当Digital Key framework被选中，设备应当按照表5-3返回数据。
钥匙设备应当向车辆指示当前配对状态，可能状态有：
- 未配对
- 配对模式开始且配对口令已经输入
SELECT指令用来选择Digital Key applet实例（使用实例AID）在15.3.2.1定义。

C-APDU: 00 A4 04 00 Lc [Digital_Key_Framework_AID] 00
R-APDU: [表 5-3]90 00 

表 5-3 SELECT指令响应

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

