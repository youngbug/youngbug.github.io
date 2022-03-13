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

这次说一下U2F应用层的协议，也就是在前两篇[FIDO U2F HID协议](https://youngbug.github.io/smartcard/2015/12/15/FIDO-U2F-HID-protocol.html)和[FIDO U2F NFC协议](https://youngbug.github.io/cryptography/2015/12/16/FIDO-U2F-NFC-protocol.html)通信协议之上的实现U2F应用的协议，其实就是U2F应用的智能卡APDU。

<!--more-->

## 1.U2F 消息封包

U2F协议是基于请求-响应方案的，当请求者发送一个请求消息到U2F设备中时，U2F会返回一个响应消息给请求者。在现在这一版的U2F协议中，消息的封包是基于ISO7816-4:2005 扩展APDU格式。

请求消息封包的格式如下：

CLA INS P1 P2 LC1 LC2 LC3 <request-data>

响应消息的格式如下：

<request-data> SW1 SW2

## 2.注册消息

### 2.1注册消息 U2F_REGISTER

![img](/assets/blog_image/2015/20151218001.png)

注册消息是需要使用U2F认证的网站在第一使用时，需要注册U2F设备的一个操作，注册请求消息有两部分，如上图所示：

challenge parameter[32 bytes]， challenge parameter是对客户端数据(Client Data)进行SHA-256运算得到的哈希值。Client Data是一个由FIDO客户端准备的JSON数据结构。

application parameter[32 bytes]，application parameter是请求注册的应用id的SHA-256结果。


### 2.2 注册消息的响应

如果注册成功，U2F令牌会创建一个新的密钥对。需要注意的是U2F令牌在返回注册成功的响应消息之前，需要用户确认（一般是带触点按键的，按下按键或者触点，没有按键的一般是拔插一下设备），否则令牌应该返回错误码test-of-user-presence-required。

![img](/assets/blog_image/2015/20151218002.png)

如果注册成功，返回的响应消息格式如上图所示。

reserved byte[1字节] 保留字节永远是0x05

user public key[65bytes] 椭圆曲线公钥

key handle length byte[1byte]  key handle的长度

key handle [长度根据上面handle lenght确定] U2F令牌用来区分产生的密钥对的。

attestation certificate[变长] X.509 DER格式的证书

signature ECDSA签名，签名的数据为0x00,application parameter, challenge parameter,key handle,user public key

签名依赖方可以使用attestation certificate中的公钥进行验证。依赖方也应该验证attestation certificate是可信的CA颁发的。

一旦依赖方验证了签名，它就应该保存public key和key handle，这样就可以在之后的认证操作中使用了。


## 3.认证消息

### 3.1 认证请求消息 U2F_AUTHENTICATE

![img](/assets/blog_image/2015/20151218003.png)

这个消息是U2F令牌进行认证的指令，FIDO客户端首先连接依赖方获得一个挑战值（随机数），然后构造认证请求报文下发到U2F令牌中，认证请求报文有下面5部分：

control byte[P1] 

值取0x07的时候表示check-only，U2F令牌只是简单的检查传入的key handle是不是这个令牌产生的，并且是不是给这个应用创建的。如果是，U2F令牌必须返回一个认证响应。

值取0x03时，表示强制用户出示并签名，U2F令牌执行一个真实的签名并响应一个认证应答消息（好别扭）


challenge parameter[32bytes] Client Data的SHA-256哈希数据

application parameter[32bytes]  是请求注册的应用id的SHA-256结果

key handle length byte[1 byte] key handle的长度

key handle[变长]注册时获得的key handle

### 3.2 注册指令的响应

U2F令牌如果发现key handle不是自己创建的，直接Bad Key Handle错误。

如果认证成功，返回响应如下

![img](/assets/blog_image/2015/20151218004.png)

U2F令牌在处理完认证请求消息后，返回上面的消息。

user presence byte[1 byte]bit0设置为1，表示用户确认过了（这一版的协议不明确的要求在进行认证请求是用户确认的方式）。这个字节剩下的bit，FIDO保留以后使用，现在都设置为0即可。

counter[4bytes]一个大段的计数器，U2F令牌每次认证都增加一次。

signature ECDSA签名数据，对app para,user presence,counter,challenge para进行签名。
