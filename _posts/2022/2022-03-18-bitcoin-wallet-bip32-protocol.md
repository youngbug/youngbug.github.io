---
layout: post
title: 比特币钱包和BIT32规范
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: SmartCard
tags: [cryptography, smartcard, bitcoin]
excerpt_separator:  <!--more-->
---

比特币钱包涉及钱包程序和钱包文件。钱包程序创建公钥来支付比特币(satoshis)，并使用对应的私钥来花掉比特币。钱包文件保存私钥和其他与钱包程序相关的交易信息(可选)。

## 钱包程序 Wallet Programs

允许接受和支付比特币是钱包软件的唯一功能，但是一个特定的钱包程序不需要同时做这两件事，两个钱包程序可以一起工作，一个程序分发公钥来接收比特币，一个程序进行交易签名来支付这些比特币。

钱包程序也需要和**peer-to-peer**网络进行交互，以从区块链中获得信息并广播出新的交易。当然，分发公钥和交易签名程序并不需要和**peer-to-peer**网络本身进行交互。

因此钱包系统(**wallet system**)就有三个必须的，但是缺可以独立的部分：一个公钥分发程序，一个签名程序，一个联网程序。

<!--more-->

>**NOTE:** 这里说的是公钥分发的通常情形。在一些情况下，P2PKH和P2SH的散列值将被分发来代替公钥的分发，实际的公钥只有在他们控制的output被支付时才分发。    
>上面和下面说的输出outputs，通常就是指 未使用的交易输出 **unspent transaction outputs** 缩写是UTXO，就是比特币。

## 完整功能的钱包 Full-Service Wallets

最简单的钱包是一个执行三个功能的程序：
- 生成私钥，并派生对应的公钥，并在需要时分发这些公钥；
- 监控支付给这个公钥的outputs，在支付outputs时，创建交易和进行交易签名；
- 广播已经完成签名的交易。

![img](/assets/blog_image/2022/20220318001-en-wallets-full-service.svg)

