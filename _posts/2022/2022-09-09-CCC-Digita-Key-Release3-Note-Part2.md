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

