---
layout: post
title: 比特币HD钱包 BIP32协议
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: SmartCard
tags: [cryptography, smartcard, bitcoin]
excerpt_separator:  <!--more-->
---

## 密钥分散 Key derivation


### 子密钥分散函数 Child key derivation (CKD) functions

给了一个父扩展密钥(**parent extended key**)和一个索引(**index**)i，可以计算出对应的子扩展密钥(**child extended key**)。算法如何计算取决于子密钥是加强密钥(**hardened key**)还是普通密钥(**normal key**),一般是判断索引的范围$i \ge 2^{31}$，并不考虑是公钥或者私钥。

#### Private parent key → private child key

函数$CKD_{priv}((k_{par},c_{par}),i) \rightarrow (k_i,c_i) $通过父扩展私钥计算子扩展私钥：

- 检查 $i \ge i^{31}$(子密钥是否是加强密钥).
    
    - 如果是(加强密钥): 令$I=HMAC-SHA512(Key=C_{par}, Data=0x00 \big| \big| ser_{256}(k_{par})\big| \big|ser_{32}(i))$。(注意：私钥要填充0x00使长度变为33字节长)
    - 如果不是(普通密钥):令$I=HMAC-SHA512(Key=C_{par}, Data=ser_p(point(k_{par}))\big| \big|ser_{32}(i))$。
- 把$I$分割成两个32字节的序列$I_L,I_R$。
- 返回子密钥$k_i=parse_{256}(I_L)+k_{par} \ (mod \ n)$。
- 返回 chain code $c_i=I_R$。
- 在$parse_{256}(I_L) \ge n $ 或者 $k_i=0$ 的情况下，密钥的结果是非法的，i应该取下一个取值再生成一次密钥。(注意：这个概率小于$1 / 2^{127}$)。

#### Public parent key → public child key

函数$CKD_{pub}((K_{par},c_{par}),i) \rightarrow (K_i, c_i)$通过父扩展公钥计算子扩展公钥。这个函数只定义了非加强子密钥。

- 检查 $i \ge 2{31}$(子密钥是是否是加强密钥)。
    - 如果是(加强密钥): 返回失败
    - 如果不是(普通密钥)：令$I=HMAC-SHA512(Key=c_{par},Data=ser_p(K_{par})\big| \big|ser_{32}(i))$。
- 把$I$分割成两个32字节序列$I_L, I_R$.
- 返回子密钥$K_i = point(parse_{256}(I_L))+K_{par}$。
- 返回chain code $c_i=I_R$。
- 在$parse_{256}(I_L) \ge n $ 或者 $k_i$是无穷的情况下，密钥的结果是非法的，i应该取下一个取值再生成一次密钥。

#### Private parent key → public child key

函数$N((K,c)) \rightarrow (K,c)$使用对应的扩展私钥计算扩展公钥(在neutered版本，删除了交易签名的能力)。

- 返回密钥$K=point(k)$。
- 返回chain code $c$就是参数传入的chain code。

使用父私钥计算子公钥：

- $N(CKD_{priv}((k_{par},c_{par}),i))$(总是这样计算)
- $CKD_{pub}(N(k_{par},c_{par}),i)$(只在非加强子密钥情况下这样计算)

在非加强密钥的情况下他们是等效的(在只给定父密钥而不需要知道私钥的情况下，就可以分散得到子公钥)。不用非加强密钥一般是因为安全原因。

#### Public parent key → private child key

这个不可能。

### 密钥树 The key tree

下一步就是级联的多个CKD来构造密钥树了。我们从主扩展密钥(master extended key)m这个根开始。