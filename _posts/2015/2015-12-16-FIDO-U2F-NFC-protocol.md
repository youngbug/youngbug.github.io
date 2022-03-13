---
layout: post
title: FIDO U2F设备的NFC协议
time: 2015年12月16日 星期三
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: Cryptography
tags: [cryptography, smartcard, nfc, fido]
excerpt_separator:  <!--more-->
---

本来计划写U2F Raw Message的内容的，但是发现FIDO联盟在2015年5月份发布的最新的U2F规范中，增加了NFC协议，所以先写下NFC的协议吧。

FIDO U2F NFC的协议其实非常简单，就是定义了一下FIDO U2F的AID和APDU的规范。

## 1.协议简介

FIDO客户端和认证设备之间通过NFC进行通讯，过程如下：

（1）客户端发送选择applet指令

（2）认证设备返回成功

（3）客户端发送操作指令（注册、认证）

（5）认证设备返回响应数据或者错误


## 2.封包的问题

U2F NFC协议不需要对消息做任何额外的封包操作（比如USB HID协议，需要对消息进行封包一样）。消息只需要按照文档U2F Raw Message中的定义直接发送到认证设备即可。


## 3.APDU的长度

部分响应数据可能比较长，一条短APDU不能传完，所以U2F 认证设备必须按下面的规则应答：

如果请求指令是扩展长度，认证设备的应答必须使用扩展APDU格式
如果请求指令不是扩展长度，认证设备的应答必须使用ISO7816-4 APDU链，比如：
