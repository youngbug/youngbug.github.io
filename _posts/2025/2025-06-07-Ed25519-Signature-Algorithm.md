---
layout: post
title: Ed25519数字签名算法
time: 2025年6月7日
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: Cryptography
tags: [Cryptography,algorithm]
excerpt_separator:  <!--more-->
---
Ed25519是基于Curve25519椭圆曲线的数字签名密码算法，是一种高效、安全的 数字签名算法，由Daniel J. Bernstein等人于2011年在论文[High-speed high-security signatures](https://eprint.iacr.org/2011/368.pdf)提出，算法如论文标题所述，它是一种快速的签名算法。它基于Edwards曲线，使用的是Twisted Edwards曲线Edwards25519，结合了强哈希函数（SHA-512和抗侧信道技术（常数时间实现）。各国政府和汽车OEM最新的网络安全规范开始要求使用更大的密钥尺寸和Ed25519算法。  

```
说明：EdDSA在RFC8032中被标准化，定义了两个方案：Ed25519和Ed448。Ed448采用的哈希函数位SHAKE256（SHA3），曲线为Curve448。
```

## Ed25519使用的曲线定义  
Ed25519签名算法使用的曲线是一种扭曲爱德华兹曲线（Twisted Edwards curve），称为Edwards25519:  

$E:-x^2+y^2=1+dx^2y^2 \ (mod\ p)$  

更常见写法是：  

$E:x^2+y^2=1+dx^2y^2$  

这是定义在有限域$F_p$上，其中：  

- **素数域：**

$p=2^{255}-19$  

这是一个255位的素数，便于高效实现，特别是在模乘、约减等操作。
