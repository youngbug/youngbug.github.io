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
    
    - 如果是(加强密钥): 令$I=HMAC-SHA512(Key=C_{par}, Data=0x00$ &#124;&#124; $ser_{256}(k_{par}) $&#124; &#124;$ser_{32}(i))$。(注意：私钥要填充0x00使长度变为33字节长)
    - 如果不是(普通密钥):令$I=HMAC-SHA512(Key=C_{par}, Data=ser_p(point(k_{par}))$&#124;&#124;$ser_{32}(i))$。
- 把$I$分割成两个32字节的序列$I_L,I_R$。
- 返回子密钥$k_i=parse_{256}(I_L)+k_{par} \ (mod \ n)$。
- 返回 chain code $c_i=I_R$。
- 在$parse_{256}(I_L) \ge n $ 或者 $k_i=0$ 的情况下，密钥的结果是非法的，i应该取下一个取值再生成一次密钥。(注意：这个概率小于$1 / 2^{127}$)。

#### Public parent key → public child key

函数$CKD_{pub}((K_{par},c_{par}),i) \rightarrow (K_i, c_i)$通过父扩展公钥计算子扩展公钥。这个函数只定义了非加强子密钥。

- 检查 $i \ge 2{31}$(子密钥是是否是加强密钥)。
    - 如果是(加强密钥): 返回失败
    - 如果不是(普通密钥)：令$I=HMAC-SHA512(Key=c_{par},Data=ser_p(K_{par})$&#124;&#124;$ser_{32}(i))$。
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

下一步就是级联的多个CKD来构造密钥树了。我们从主扩展密钥(master extended key)m这个根开始。通过使用多个$i$值对$CKD_{priv(m,i)}$求值，我们可以得到大量一级派生节点(level-1 derived nodes)。这些派生节点也是扩展密钥，可以使用$CKD_{priv}$继续正常进行分散计算。

使用简化符号时，我们把$CKD_{priv}(CDK_{priv}(CKD_{priv}(m,3_H),2),5)$ 写为 $m/3_H/2/5$。公钥也是一样的，我们把$CKD_{pub}(CKD_{pub}(CKD_{pub}(M,3),2),5)$ 写为$M/3/2/5$。这就有了以下的特性：

- $N(m/a/b/c) = N(m/a/b)/c = N(m/a)/b/c = N(m)/a/b/c = M/a/b/c$
- $N(m/a_H/b/c)=N(m/a_H/b)/c=N(m/a_H)/b/c$

然而$N(m/a_H)$不能再写为$N(m)/a_H$，因为后者不可能。

树上的每个叶子节点都对应一个实际的密钥，同时内部节点对应于从他们派生出来的密钥的集合。叶子节点的chain code被忽略，只有嵌入他们的公私密钥是有意义的。因这种构建，知道了扩展私钥就可以重建所有的后代私钥和公钥，知道一个扩展公钥允许重建所有后代的非强化公钥。

### 密钥标志 Key identifiers

扩展公钥可以通过序列化后的ECDSA公钥K的Hash160(RIPEMD160 after SHA256)来识别，忽略掉chain code。这个与传统比特币地址使用的数据完全一致。不建议使用Base58格式来表示该数据，因为它可能会被当做一个地址(钱包软件接受支付并不需对密钥链的支持)。

密钥标志的前32 bits称作密钥指纹(**key fingerprint**)

### 序列化格式 Serialization format

扩展公钥和私钥使用下面的流程序列化：

- 4 bytes：版本字节(mainnet:0x0488B21E public, 0x0488ADE4 private; testnet: 0x043587CF public, 0x04358394 private)
- 1 byte: 深度：0x00是主节点，0x01是第一级分散密钥(level-1 derived keys) $\dots$
- 4 bytes: 父密钥的指纹(主密钥的值为0x00000000)
- 4 bytes: 子代号码(child number)。This is ser32(i) for i in xi = xpar/i, with xi the key being serialized. (0x00000000 if master key)
- 32 bytes：chain code
- 33 bytes：公钥或者私钥数据(公钥$ser_p(K)$,私钥$0x00$&#124;&#124;$ser_{256}(k)$)

这个78字节的结构可以像其他比特币数据一样使用Base58编码