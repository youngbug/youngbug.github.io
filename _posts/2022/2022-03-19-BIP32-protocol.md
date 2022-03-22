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

使用简化符号时，我们把 $CKD_{priv}(CDK_{priv}(CKD_{priv}(m,3_H),2),5) 写为 m/3_H/2/5 。公钥也是一样的，我们把CKD_{pub}(CKD_{pub}(CKD_{pub}(M,3),2),5) 写为M/3/2/5$。这就有了以下的特性：

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

这个78字节的结构可以像其他比特币数据一样使用Base58编码，首先添加32 bits的校验和(通过两次SHA256得到)，然后转换成Base58表示。这将生成一个最多112个字符的Base58字符串。因为版本字节的选择，Base58的表示中，mainnet将以xprv或者xpub开始，testnet将以tprv或者tpub开始。

注意，父节点的指纹只用来作为软件检测父子节点的快速方法，而软件必须愿意处理冲突。在内部，可以使用完整的160 bits标识。

当导入经过序列化的扩展公钥时，实现的过程必须验证公钥中的X坐标是否在椭圆曲线的点上。如果不在，扩展公钥就是非法的。

### 生成主密钥 Master key generation

可能的扩展密钥对总数差不多是$2^{512}$ 个，但是生成的密钥长度只有256bits，提供只有一半的安全性。因此主密钥不是直接生成的，而是通过一个比较短的种子值生成的。

- 通过一个随机数生成一个选定长度种子字节序列S(在128~512 bits之间；建议256 bits)
- 计算$I=HMAC-SHA512(Key="Bitcoin seed", Data=S)$
- 把$I$分割为两个32字节，$I_L,I_R$
- 使用$parse_{256}(I_L)$作为主密钥(master secret key)，$I_R$作为主chain code

在$parse_{256}(I_L)=0$或者$parse_{256}(I_L) \ge n$，主密钥是非法的。

## 钱包结构 Wallet structure

前面规定了密钥树和他们的节点。接下来是在这个树上添加一个钱包结构。这里的布局只是默认的，为了兼容鼓励客户端去兼容它，但并不是所有的特性都被支持。

### 默认的钱包布局 The default wallet layout

一个HD钱包由多个账户(**accounts**)组成。账号是编号的，默认的账号("")编号是0.客户端并不需要支持超过1个账户，如果不支持，它只支持默认账户。

每个账户都由两个密钥对链组成：一个内部和一个外部的密钥。外部密钥链用于生成新公钥地址，同时内部密钥链用于其他操作(修改地址，产生地址，任何不需要沟通的操作)。不支持单独密钥链的客户端应该使用外部密钥做一切操作。

- $m/i_H/0/k$表示从主密钥m分散出来的编号i的HD钱包账户的**外部**链的第k个密钥对
- $m/i_H/1/k$表示从主密钥m分散出来的编号i的HD钱包账户的**内部**链的第k个密钥对

### 用例 Use cases

#### Full wallet sharing: m

在两个系统需要访问一个单独的共享钱包的情况下，两者都需要能够消费，一个需要分享主扩展私钥。节点可以维护一个密钥池子，池中为外链缓存N个将来使用的密钥，来监控收入。为将来使用的内链可以非常小，因为这里不会出现gaps。额外的为将来使用的密钥可以为第一次尚未使用的账户链激活--当使用时触发创建新账户。注意账户名字仍然需要手工输入，而不能通过区块链同步。

#### 审计 Audits: N(m/*)

如果审计人员需要查看所有收入和支出的列表，可以共享所有账户的扩展公钥。这将允许审计人员查看钱包中全部账户的所有进出的交易，但不能查看单个的密钥。

#### 每个办公室余额 Per-office balances: $m/i_H$

当一个公司有多个独立的办公室时，他们都能使用一个单独密钥分散出的钱包。他们允许总部维护一个可以看到所有办公室的收入和支出交易的超级钱包，甚至允许在办公室之间转移资金。

#### 经常性公对公交易 Recurrent business-to-business transactions: $N(m/i_H/0)$

在两个经营合作伙伴经常转账的情况时，一方可以使用指定账户($M/i_H/0$)的**外链**的扩展公钥作为一种"超级地址"(“super address”)，可以让频繁的交易不会(不容易)被关联起来，但是不需要每次都为支付比特币请求一个新的地址。这种方案也被矿池用作可变支付地址。

#### 不安全的收款方 Unsecure money receiver: $N(m/i_H/0)$

当一个不安全的web服务器被用来运行一个电子商务站点，它需要知道用来接受付款的公钥地址。web服务器只需要知道单个账户的**外链**扩展公钥。这意味着某人非法的web服务器的访问权最多看到所有的收款支付，但是不能盗走资金，不能(琐碎)区分付款的交易，也不能看到其他web服务器的收款交易，如果有多个的话。

### 兼容性 Compatibility

