---
layout: post
title: CCC 数字钥匙Release 3 学习笔记 车主配对 Part3
time: 2023年4月12日
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: cryptography
tags: [cryptography]
excerpt_separator:  <!--more-->
---

## 6.1 概述

车主配对由钥匙设备上的Digital Key framework来控制，通过使用APDU管理数字钥匙的配置，整个方案由智能卡芯片提供保护。

智能卡COS提供可信的根，提供信任链的起点。

车辆能够通过NFC使用数字钥匙的AID选择Digital Key applet，并通过对应的AID选择Digital Key framework。 基于选择AID，NFC控制器可以重新配置来改变和智能卡或者Digital Key framework通信，反之亦然。

新车主设备配对流或者车主设备修改不意味着有隐式的取消配对，比如，一个新钥匙设备车主配对流只改变车主的钥匙。存在的已经配对的共享/好友钥匙和车辆的公钥不会受到影响。

Digital Key applet实例应当在车主配对执行之前为可用状态。

<!--more-->

## 6.2 密钥和数据

下面定义了车主配对时用到的密钥和数据元素：

- **device.PK/device.SK**: 钥匙设备在数字钥匙创建时产生的长期密钥对。一个钥匙一对。

- **vehicle.PK/vehicle.SK**：车辆产生的密钥对。相同车辆的所有的数字钥匙这个密钥对相同。这个密钥对的生命周期，由主机厂规定，不在CCC数字钥匙规范里规定。

- **配对口令**：输入到钥匙设备的SPAKE2+配对口令。由主机厂账户提供的UTF8编码的8位数字（0-9），用于车主认证。

- **Kenc**：派生出的对称密钥（通过SPAKE2+共享密钥生成），用于加密保密的指令和响应负载。

- **Kmac**：派生的对称密钥（通过SPAKE2+共享密钥生成），用于计算指令C-APDU的MAC。

- **Krmac**：派生的对称密钥（通过SPAKE2+共享密钥生成），用于计算响应R-APDU的MAC。

## 6.3 车主配对实现

这部分描述了车主配对流程中不同阶段，包括：

- **阶段0**：准备（见6.3.1）

- **阶段1**：在车辆和钥匙设备上初始化配对过程（见6.3.2）

- **阶段2**：同NFC读卡器的的第一个会话（见6.3.3）

- **阶段3**：同NFC读卡器的第二个会话（见6.3.4）

- **阶段4**：结束配对过程（见6.3.5）

车主配对流程包括两个会话：第一个会话（阶段2）与Digital Key framework执行；第二个会话（阶段3）与Digital Key applet执行，如图6-1.在车主配对阶段2，车辆配置了是去主机厂服务器在线检索防盗令牌（immobilizer token）还是从车辆内检索。如果配置为在线检索，车主防盗令牌不会再第二个会话中通过NFC读卡器传输，但在配对时或者结束后在钥匙追踪请求/响应（Key Tracking Request/Response）至/来自 车辆主机厂服务器和车主钥匙设备厂家服务器传输。图6-1描述了车主配对过程（防盗令牌在车内检索）

第一个会话包括两个NFC交易。第一个交易是协商协议版本，执行SPAKE2+，并传输所有的钥匙创建数据到钥匙设备。第二个交易，在钥匙设备创建数字钥匙之后执行，提供创建的证明和证书链给车辆。

![img](/assets/blog_image/2023/202304120001-figure-6-1.png)

### 6.3.1 阶段0：准备

#### 6.3.1.1 设备准备

确保钥匙设备已经安装了Digital Key applet（CCC的规范里没有规定，这个是Global Platform里的内容了，参考GP规范即可），在车主配对前为每个主机厂创建了一个Instance CA。Digital Key framework已经由主机厂合作伙伴更新。

Instance CA证书由Digital Key framework获得。

参考Listing 15-15 Instance CA证书的描述。所有签名通过18.4.10的描述产生。

#### 6.3.1.2 车辆准备

基于SPAKE2+协议创建车主配对安全通道，参见18.4.1.这个协议绑定设备到用户，可以防窃听和对抗MITM攻击。

在车主配对过程执行前，车辆主机厂服务器为钥匙设备产生配对口令，验证车辆的口令verifier描述在Listing 18-1，通过主机厂服务器安全通道发送给车辆verifier和盐值，如图6-2的第一步。

### 6.3.2 阶段1：初始化配对过程

在车辆端，配对过程初始化是主机厂的责任。适合的车主配对准备需要被满足，比如存在一个密钥（还是钥匙？）容器？

![img](/assets/blog_image/2023/202304120002-figure-6-2.png)

只要没有车主配对，车辆要么由用户设置进入配对模式，要么通过车辆控制台上的NFC读卡器尝试选择framework AID。

开始车主配对模式（图6-2的步骤3），钥匙设备接收到口令，要么通过用户输入，要么通过URL（参考6.3.7），要么通过一个API直接从主机厂APP获得。

### 6.3.3 阶段2：同NFC读卡器的第一个会话

第一个NFC会话包括2个不同的NFC交易。

第一个NFC交易协商协议版本（见6.3.3.8），车辆和钥匙设备互相认证，使用SPAKE2+建立安全通道，传输所有钥匙创建数据给钥匙设备，参见图6-3。

![img](/assets/blog_image/2023/202304120003-figure-6-3.png)

SPAKE2+创建一个对称会话密钥用来在汽车和设备之间建立一个安全通道。

在第二个NFC交易开开始之前，设备创建了设备密钥。

在第二个NFC交易，汽车从设备中读取钥匙创建数据，验证这些数据，如果成功，存储设备公钥。NFC复位在每次交易之后执行。

紧接着的步骤或者事件顺序都是发生在NFC会话中：

#### 6.3.3.1 步骤1：选择Digital Key Framework

当设备和汽车的NFC读卡器通信时，汽车应当使用选择指令通过AID选中Digital Key Framework。如果车俩选择Digital Key Applet AID在选择Digital Key framework AID之前，设备应当对选择Digital Key Applet的指令响应状态字6A82。设备应当通过选择AID的响应返回给汽车所有支持的SPAKE2+版本和所有Digital Key applet协议版本。决定双方用哪个版本的SPAKE2+（用来创建安全通道）和使用哪个版本的Digital Key applet协议（用于钥匙分享）。

选择应用指令定义在 5.1.1

#### 6.3.3.2 步骤2和2a：SPAKE2+ 交易

汽车应当选择并发送给设备要使用的SPAKE2+协议版本和支持的Digital Key applet协议版本列表。

SPAKE2+交易在[10]描述。

![img](/assets/blog_image/2023/202304120004-figure-6-4.png)

