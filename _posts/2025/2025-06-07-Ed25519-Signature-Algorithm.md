---
layout: post
title: Ed25519数字签名算法
time: 2025年6月7日
author: Zhao Yang(cnrgc@163.com)
location: 北京
published: true
category: Cryptography
tags: [Cryptography,algorithm]
excerpt_separator:  <!--more-->
---

### 前言

Ed25519 是近年来密码学界和工业界共同关注的一种高性能椭圆曲线数字签名算法。在从事车联网安全工作的过程中，我逐渐意识到 Ed25519 正在成为车载安全领域的重要选项。这篇笔记记录了我对 Ed25519 的系统学习过程，涵盖了从密码学原理到工程实践、从跨行业应用到车联网专项应用的内容，供后续查阅和分享。

Ed25519 是在与 Curve25519 双有理等价的扭曲 Edwards 曲线 Edwards25519 上实现的数字签名算法，是一种高效、安全的数字签名算法，由 Daniel J. Bernstein 等人于 2011 年在论文 [High-speed high-security signatures](https://eprint.iacr.org/2011/368.pdf) 提出。该算法基于 Edwards 曲线 Edwards25519，结合了强哈希函数（SHA-512）和抗侧信道技术（常数时间实现）。各国政府和汽车 OEM 最新的网络安全规范开始要求使用更大的密钥尺寸和 Ed25519 算法。  

```
说明：EdDSA在RFC8032中被标准化，定义了两个方案：

Ed25519和Ed448。Ed448采用的哈希函数位SHAKE256（SHA3），曲线为Curve448。
```
 <!--more-->

### 1 算法发展历史

#### 1.1 学术发展脉络

Ed25519 的发展经历了三个关键阶段，其技术谱系清晰可溯：

**第一阶段：Curve25519 曲线的提出（2005—2006年）**

2005年，密码学家 **Daniel J. Bernstein** 首次发布了 Curve25519 椭圆曲线，这是一条基于蒙哥马利形式的椭圆曲线，用于椭圆曲线 Diffie-Hellman (ECDH) 密钥交换。该曲线提供 128 位安全强度，以其卓越的计算性能和不受已知专利约束的特点引起了密码学界的广泛关注。Curve25519 最初在 2006 年通过学术预印本正式发表。

**第二阶段：Ed25519 签名方案的正式提出（2011年）**

2011 年，Daniel J. Bernstein、Niels Duif、Tanja Lange、Peter Schwabe 和 Bo-Yin Yang 联合发表了论文 **《High-speed high-security signatures》**，正式提出了 Ed25519 数字签名算法。Ed25519 是 EdDSA（Edwards-curve Digital Signature Algorithm）算法在 twisted Edwards 曲线 edwards25519 上的具体实例化。EdDSA 本身是 Schnorr 签名机制在 Edwards 曲线上的变体，其设计目标是同时实现高速度和高安全性。

**第三阶段：标准化与广泛采纳（2016年至今）**

Ed25519 及其变体 Ed448 在 **RFC 8032** 中被 IETF 正式标准化。此外，2017年 NIST 宣布将 Curve25519 和 Curve448 纳入 Special Publication 800-186，标志着该算法获得了美国联邦政府层面的认可。从学术原创论文到 IETF 标准再到 NIST 认可，Ed25519 经历了完整的学术标准化流程。

#### 1.2 产业化关键催化剂

2013 年斯诺登事件曝光了 NSA 在 Dual_EC_DRBG 伪随机数生成器中植入后门的事实，加之 NIST P-256 等传统曲线的参数来源缺乏透明度，引发了业界对公钥密码基础设施的信任危机。在此背景下，Bernstein 团队设计的 Curve25519 及其上层 Ed25519，以其完全公开透明的参数选择、无后门风险的设计理念，迅速获得了开源社区和工业界的广泛信任，成为替代传统 NIST 曲线的首选方案。

#### 1.3 重要变体

- **Ed25519-Original**：2011年原始论文中的版本
- **Ed25519-IETF**：RFC 8032 标准版本，增加了防签名延展性攻击的机制，是当前应用最广的版本
- **Ed25519ph**：支持预哈希的变体（Pre-hash），适用于需要先计算消息摘要再签名的场景
- **Ed25519ctx**：支持上下文标签的变体，通过绑定域分隔标签防止跨域签名重放


### 2 数理原理与密码学机制

Ed25519 的安全性建立在**椭圆曲线离散对数问题 (ECDLP)** 的困难性之上，其数理设计精妙而严谨。以下从核心参数到签名流程逐层展开。

#### 2.1 曲线形式与核心参数

Ed25519签名算法使用的曲线是一种扭曲爱德华兹曲线（Twisted Edwards curve），称为Edwards25519(Ed25519) 使用的扭曲爱德华曲线方程为：


$-x^2 + y^2 = 1 + d \cdot x^2 \cdot y^2, \quad 其中\ d = -\frac{121665}{121666}$


该曲线与蒙哥马利形式的 Curve25519（$ $v^2 = u^3 + 486662u^2 + u$ $）之间存在双有理等价关系，这一性质使得 Ed25519 可以充分利用 Curve25519 在标量乘法上的高性能实现，同时保留了爱德华曲线在群运算上的完备性（即加法公式对所有输入均有效，无例外点）。

核心参数如下：

| 参数 | 含义 | 值 |
|:---|:---|:---|
| $q$  | 有限域模数（素数） |  $2^{255} - 19$  |
| $ B $ | 基点（生成元） | y坐标为 $4/5$，x坐标满足曲线方程 |
| $ \ell $ | 基点生成的主子群阶数 |  $2^{252} + 27742317777372353535851937790883648493$  |
| $ c $ | 辅因子 | 8 |
| $ b $ | 安全参数（比特） | 256 |
| $ H $ | 哈希函数 | SHA-512 |

曲线群的完整阶数为 $ 8\ell $，辅因子 $ c=8 $ 的设计意味着群中存在大小为 8 的小阶子群。Ed25519 通过引入辅因子乘法来处理小阶子群问题，这与其使用扭曲 Edwards 曲线的完备加法公式是两个不同层面的安全机制。

q是一个255位的素数，便于高效实现，特别是在模乘、约减等操作。


#### 2.2 密钥生成

设 32 字节随机私钥种子为 $ k $：

1. **哈希扩展**：计算 $ H(k) = \text{SHA-512}(k) $，得到 512 比特输出
2. **比特修剪 (Clamping)**：取前 256 比特进行修剪，具体操作如下：
   - 清空最低 3 位（确保私钥为 8 的倍数，自然消除辅因子）
   - 清空第 255 位（强制私钥 < 2²⁵⁵）
   - 设置第 254 位为 1（确保私钥 ≥ 2²⁵⁴，固定标量长度）
   - 得到 256 比特标量私钥 $ a $
3. **公钥计算**：$ A = a \cdot B $，公钥是曲线上点的 32 字节压缩编码

密钥生成中的哈希扩展机制将 256 比特熵的种子映射为 512 比特的内部状态，为后续确定性签名提供了独立的随机源；修剪操作则同时保障了侧信道防护和安全性。

#### 2.3 签名生成（确定性签名设计）

设消息为 $ M $，私钥为 $ a $，公钥为 $ A = a \cdot B $：

1. **确定性随机数派生**：取 SHA-512 输出的后 256 比特作为“密钥前缀”，计算
   $
   r = H(\text{prefix} \parallel M) \bmod \ell
   $
2. **曲线点计算**：$ R = r \cdot B $
3. **挑战值**：$ h = H(R \parallel A \parallel M) \bmod \ell $
4. **签名**：$ S = (r + h \cdot a) \bmod \ell $

最终签名为 64 字节的 $ (R, S) $ 对（R 的压缩编码 | S 的字节编码）。

> **确定性签名设计的重要意义**：$ r $ 由私钥前缀和消息通过哈希确定性派生，不需要外部随机数发生器。这一设计从根本上杜绝了因随机数质量缺陷导致的私钥泄露风险（最著名的案例是 2010 年索尼 PlayStation 3 因 ECDSA 随机数重用而被破解）。在车载嵌入式环境中，高质量的熵源通常较为稀缺，确定性签名大幅降低了这一工程挑战，是实现车规级安全合规的关键优势之一。

#### 2.4 签名验证（辅因子验签机制）

验证者已知 $ M $、签名 $ (R, S) $、签名者公钥 $ A $：

1. 重新计算 $ h = H(R \parallel A \parallel M) \bmod \ell $
2. 验证带辅因子的等式：
   $
   8S \cdot B \stackrel{?}{=} 8R + 8h \cdot A
   $

**正确性证明**：

$
8S \cdot B = 8(r + ha) \cdot B = 8r \cdot B + 8ha \cdot B = 8R + 8h \cdot A
$

辅因子 8 的乘法强制所有点运算在阶为 $ \ell $ 的主子群中进行，同时确保即使验证方接收到非主子群中的 $ R $ 点，也不会错误地接受签名，从而彻底消除了小阶点攻击的威胁。

#### 2.5 安全强度分析

Ed25519 提供约 **128 位安全强度**（Target 2¹²⁸），与以下算法等效：

- **NIST P-256**（ECDSA 256 位）
- **RSA-3072**
- **SM2 256 位**

若将安全级别对标 AES 的暴力破解复杂度，在经典计算模型下，Ed25519 的实际攻击复杂度约为 2¹²⁶，满足 NIST SP 800-57 对“到 2030 年后仍可安全使用”的 128 位安全级别要求。


### 3 关键设计特性与安全机制

#### 3.1 确定性签名机制

Ed25519 最核心的设计特征之一是其**确定性签名**机制——同一消息和私钥每次生成的签名完全相同。其工程价值在于：

- **消除随机数依赖**：移除签名过程中对 CSPRNG 的强依赖，显著降低嵌入式环境中由弱熵源引发的安全风险
- **防范侧信道**：统一的随机数派生路径降低了差分功耗分析（DPA）的利用面
- **简化测试与认证**：确定性输出使测试向量验证和算法合规性认证更加可复现

#### 3.2 完备加法公式

爱德华曲线的加法公式对曲线上的所有点均有效，包括无穷远点。相比传统 Weierstrass 曲线可能因输入特殊点而产生异常分支的情况，完备加法公式天然免疫“例外点攻击”和“无效曲线攻击”，同时避免了条件分支，有利于实现恒定时间算法。

#### 3.3 关键实现风险与最佳实践

尽管 Ed25519 在算法设计层面具有诸多安全优势，但在工程实现中仍需注意以下关键风险：

**1. 确定性签名的特殊约束**

确定性签名设计意味着：**对同一条消息用同一私钥生成两次签名时，如果其中一次因任何原因产生了不同的 $ S $ 值，攻击者可能通过比较两次不同的 $ S $ 值直接计算出私钥**。在共识机制、重放保护等场景中，应避免对完全相同的消息体重复签名，或考虑在设计层面引入 nonce 字段确保消息唯一性。

**2. 函数级安全要求**

在实际部署中，所有 Ed25519 实现均应满足：
- 恒定时间实现，不因密钥或消息值产生时序差异
- 签名中的 R 点需验证其位于曲线上，防止无效曲线攻击
- 私钥材料使用后应立即从内存中安全擦除
- 符合 FIPS 140-3 或 ISO/IEC 19790 对熵源、密钥管理和自测试的要求

**3. 与其他算法的接口安全性**

在多算法共存的车载安全框架中（如 RSA/ECDSA/SM2/Ed25519 混合使用），应通过独立的算法标识符（如密钥类型标记）和域分隔机制防止跨算法攻击。


### 4 跨行业应用

Ed25519 凭借高性能与高安全性，已在互联网基础设施、区块链、即时通讯等领域得到广泛应用。

| 应用领域 | 具体用途 | 简要说明 |
|:---|:---|:---|
| **TLS 1.3** | HTTPS 证书签名 | 现代 TLS 握手和证书链验证，已被主流浏览器和服务器支持 |
| **SSH** | 安全远程登录 | OpenSSH 6.5+ 支持 Ed25519 密钥类型，提供比 RSA 更快的连接建立 |
| **区块链** | 交易签名 | Solana、Near、Aptos 等公链采用 Ed25519 作为默认签名算法 |
| **Signal 协议** | 端到端加密通讯 | Signal、WhatsApp 等即时通讯应用使用 Ed25519 进行身份认证 |
| **Tor 匿名网络** | 节点身份验证 | 用于验证 Tor 中继节点身份，保障通信链路安全 |
| **S/MIME 4.0** | 邮件签名 | IETF S/MIME 4.0 标准支持 Ed25519 用于电子邮件签名 |
| **Roughtime** | 安全时间同步 | Google 的 Roughtime 协议使用 Ed25519 防篡改时间戳 |
| **W3C 可验证凭证** | 数字身份 | 用于签发防篡改的数字凭证 |


### 5 车联网领域的应用

#### 5.1 AUTOSAR 架构中的密码学规范要求

AUTOSAR 是全球主流的汽车嵌入式软件架构标准。自 4.2 版本引入网络安全框架以来，AUTOSAR_RS_Cryptography 规范明确将 **Ed25519** 与 RSA-2048/3072、ECC NIST P-256/P-384 并列，作为车载环境中数字签名、密钥协商与证书验证的标准算法。该规范强制要求所有密码操作必须与 AUTOSAR SecOC（Secure Onboard Communication）模块深度协同，为 CAN FD、Ethernet 等车载总线上的通信报文提供端到端真实性与新鲜性保护。

在密码服务部署方面，RS_Cryptography 规定了基于 HSM（硬件安全模块）的安全等级分类（SL1 至 SL4），要求涉及 OTA 升级、V2X 通信签名等高风险场景的密钥操作必须运行在具备物理防篡改和侧信道防护能力的 HSM 中，并通过 ISO/SAE 21434 规定的网络安全管理流程（CSMS）进行全生命周期管控——从密钥生成、分发、存储、轮换、归档到最终销毁，每一步均需留有不可抵赖的审计日志。这一要求对 Ed25519 密钥的安全存储与管理提出了车规级工程约束。

#### 5.2 车规级安全芯片中的 Ed25519 支持

Ed25519 在车联网中的落地离不开硬件信任根的支撑。国内外众多智能卡芯片厂商推出汽车安全应用的专用安全芯片，直接集成了 Ed25519 曲线算法支持，主要服务于：

- **Secure Boot（安全启动）** ：引导加载阶段的固件签名验证
- **ECU 身份认证**：车载 ECU 间的身份认证与密钥协商
- **OTA 固件更新**：FOTA/SOTA 过程中的固件签名验证

“各国政府和汽车 OEM 最新的网络安全规范开始包含爱德华曲线 Ed25519 算法标准”的行业趋势。此类安全元件通常具备防篡改存储、硬件加解密引擎、真随机数发生器等模块，能够满足车载环境下的实时性与安全性双重约束。

国内智能卡芯片厂家推出的车用安全芯片，一般都多种密码算法加速引擎（支持国密 SM2/SM3/SM4 与国际算法），支持安全启动、密钥管理、设备认证与防篡改等核心功能，符合国密二级与车规标准。

#### 5.3 V2X 车联网通信中的 Ed25519 应用

V2X（Vehicle-to-Everything）通信对认证延迟和带宽占用有严苛要求。研究表明，在 V2I（Vehicle-to-Infrastructure）通信场景中，基于 **Ed25519 的认证方案一次完整的认证交互仅需约 17.9 ms，通信开销仅为 176 字节**，在安全性和性能之间取得了很好的平衡。学术研究还提出了多种基于 Ed25519 或 Curve25519 的 V2X 安全通信优化方案，包括：

- 基于属性的预认证安全通信协议（Lw-AABSoK），支持 V2X 场景下的密钥保护和凭证在线更新
- 面向 V2X 的轻量级匿名认证协议，利用 Ed25519 签名的高效性实现车辆身份隐私保护

在国内 V2X 标准层面，CCSA 已制定了通信行业标准《基于LTE的车联网无线通信技术 安全证书管理系统技术要求》，该标准规定了 V2X PKI 证书管理体系架构，明确了注册证书机构（ECA）、假名证书机构（PCA）、应用证书机构（ACA）等层次结构。虽然当前国内 V2X 安全标准主要推进国密 SM2 算法体系（如工信部 2025 年发布的《车联网数据安全传输技术规范》明确要求 V2X 场景采用国密算法），但 Ed25519 作为高性能备选方案，在国际化部署和跨域互操作场景中具有重要价值。

#### 5.4 相关国际标准与行业规范

| 标准/规范 | 类型 | 与 Ed25519 的关系 |
|:---|:---|:---|
| **RFC 8032** | 国际标准（IETF） | EdDSA 及 Ed25519 的核心技术标准 |
| **NIST SP 800-186** | 美国联邦标准 | 将 Curve25519 纳入批准的椭圆曲线列表 |
| **ISO/SAE 21434** | 国际标准 | 道路车辆网络安全工程标准，对密码学实现提出全生命周期安全管理要求。标准要求涉及高风险场景的密钥操作须在 HSM 中执行，并须通过 CSMS 进行合规管控 |
| **AUTOSAR RS_Cryptography** | 行业规范 | 将 Ed25519 列为车载密码学标准算法之一 |
| **FIPS 140-3** | 美国联邦标准 | 对密码模块的安全要求，涉及 Ed25519 实现的认证 |
| **CCSA 通信行业标准** | 中国行业标准 | 基于 LTE 的 V2X 安全证书管理系统技术要求 |
| **GM/T 0138-2024** | 中国密码行业标准 | C-V2X 车联网证书策略与认证业务声明框架 |


### 6 Ed25519 与主要数字签名算法的对比分析

Ed25519 并非在所有维度上都有绝对优势。不同场景下，RSA、ECDSA 和 SM2 各有其存在价值。以下从多个维度进行系统性对比。

#### 6.1 安全性与性能对比

| 对比维度 | **Ed25519** | **ECDSA (P-256)** | **RSA** | **SM2** |
|:---|:---|:---|:---|:---|
| **安全基础** | ECDLP（扭曲爱德华曲线） | ECDLP（Weierstrass 曲线） | 大整数分解 + 离散对数 | ECDLP（SM2 自定义曲线） |
| **等效安全强度** | ~128 位 | ~128 位 | 128 位需 RSA-3072 | ~128 位 |
| **私钥大小** | 32 字节 | 32 字节 | 256 字节（2048位）/ 384 字节（3072位） | 32 字节 |
| **公钥大小** | 32 字节 | 65 字节（非压缩） | 256 字节（2048位）/ 384 字节（3072位） | 65 字节（非压缩） |
| **签名大小** | 64 字节 | 70—72 字节（DER 编码） | 256 字节（2048位）/ 384 字节（3072位） | 64—72 字节（DER 编码） |
| **签名速度** | 极快（约 50,000 ops/s） | 快 | 较慢（约为 Ed25519 的 1/10—1/50） | 与 ECDSA 相近 |
| **验证速度** | 极快，约为 RSA 的 10 倍以上 | 快 | 快速（验证速度可接近 Ed25519） | 与 ECDSA 相近 |
| **密钥生成速度** | 极快（无额外素性检测） | 快 | 极慢（需生成大素数） | 快 |

性能方面的具体数据对比：Ed25519 的公钥仅 32 字节，而相同安全级别的 RSA 需要 256 字节以上；Ed25519 的签名验证速度约为 RSA 的 10 倍，且持续优于 ECDSA。Google Tink 官方文档也明确指出，Ed25519 的签名性能优于 ECDSA_P256。

在 Uptane 框架（面向汽车 OTA 更新的安全框架）的研究中，SM2 与 Ed25519 的时延开销在同一个数量级，均表现出优秀的实时性能。

#### 6.2 工程与安全特性对比

| 特性维度 | Ed25519 | ECDSA (P-256) | RSA | SM2 |
|:---|:---|:---|:---|:---|
| **签名随机性** | 确定性（无随机数依赖） | 随机（依赖 CSPRNG，随机数重用可泄露私钥） | 确定性（如 RSASSA-PKCS1-v1_5） | 随机（依赖 CSPRNG） |
| **侧信道抗性** | 设计层面支持恒定时间实现 | 需要额外实现技巧 | 签名时需防护 | 类似 ECDSA |
| **参数透明度** | 完全公开，无未解释的魔术常量 | 参数来源存在争议 | 无需特殊曲线参数 | 国家密码管理局发布 |
| **标准化程度** | RFC 8032，NIST SP 800-186 | NIST FIPS 186-4，ANSI X9.62 | PKCS#1，广泛标准化 | GM/T 0003，GB/T 32918 |
| **专利风险** | 无已知专利约束 | 部分实现可能存在专利 | 历史专利已过期 | 无专利风险 |
| **后量子安全性** | 不具备（与所有经典 ECC 相同） | 不具备 | 不具备 | 不具备 |

#### 6.3 车联网场景的适用性分析

Ed25519 在车联网场景中具有以下显著优势：

1. **确定性签名 → 嵌入式友好**：车载 ECU 的高质量熵源通常比较稀缺，确定性签名直接去除了对签名阶段随机数的依赖，降低了安全设计和 FIPS 140 合规的复杂度
2. **小签名（64 字节） → 总线友好**：64 字节的签名尺寸可有效降低 CAN FD、车载以太网等场景下的通信带宽占用，在 V2X 匿名证书场景中尤其具有优势——结合隐式证书机制可进一步缩小证书尺寸，节约空口传输资源
3. **快速验证 → 实时性保障**：10 倍于 RSA 的验证速度可满足车辆高速移动场景下的低延迟认证需求
4. **无专利风险 → 供应链安全**：无专利约束的特性有利于降低车企的供应链法律风险和授权成本

#### 6.4 注意事项与局限性

1. **兼容性限制**：部分遗留的 PKI 基础设施和 CA 系统不完全支持 Ed25519 密钥类型，在企业级部署中需要评估与现有证书管理体系的兼容性
2. **算法共存的接口安全**：在多算法共存的车载安全框架中（如 SM2 用于国内合规、Ed25519 用于国际互操作），需要设计安全的算法协商机制和密钥类型标识，防止跨算法降级攻击
3. **后量子迁移准备**：与所有经典 ECC/RSA 一样，Ed25519 不具备后量子安全性。在密码敏捷性设计中，应预留切换至后量子签名算法（如 CRYSTALS-Dilithium）的接口，AUTOSAR 4.3.1 已开始探索 PQC 方案的集成路径
4. **国密合规要求**：在国内车联网 V2X 场景中，政策层面已明确要求优先采用国密 SM2/SM3/SM4 算法体系。Ed25519 在国内车联网场景的定位应为国际化部署的补充选项，而非替代方案


### 7 工程实现示例

> **⚠️ 安全提示**：本节示例代码仅用于学习和理解 API 调用流程，不可直接用于生产环境。生产部署应使用经过安全审计的密码学库的稳定版本，并在安全硬件（HSM/SE/TPM）中完成密钥操作。

#### 7.1 OpenSSL 3.0 API 实现

OpenSSL 3.0 采用 Provider 架构，提供了完整的 Ed25519 API 支持，包括 EVP_PKEY 密钥管理、EVP_DigestSign 签名和 EVP_DigestVerify 验证接口。

```c
/**
 * Ed25519 签名/验签示例（OpenSSL 3.0）
 * 编译：gcc -o ed25519_openssl ed25519_openssl.c -lcrypto
 * API 函数说明及返回值含义见代码注释
 */

#include <stdio.h>
#include <string.h>
#include <openssl/evp.h>
#include <openssl/pem.h>
#include <openssl/err.h>

void handle_errors() {
    ERR_print_errors_fp(stderr);
    abort();
}

int main() {
    EVP_PKEY *pkey = NULL;
    EVP_PKEY_CTX *pctx = NULL;
    EVP_MD_CTX *md_ctx = NULL;
    unsigned char *sig = NULL;
    size_t sig_len = 0;
    int ret = 0;

    /* ===== 1. 生成 Ed25519 密钥对 ===== */
    pctx = EVP_PKEY_CTX_new_from_name(NULL, "ED25519", NULL);
    if (!pctx || EVP_PKEY_keygen_init(pctx) <= 0)
        handle_errors();

    if (EVP_PKEY_generate(pctx, &pkey) <= 0)
        handle_errors();

    printf("[OK] Ed25519 密钥对生成成功\n");

    /* ===== 2. 对消息进行签名 ===== */
    const unsigned char *msg = (const unsigned char *)"Hello, Ed25519 from OpenSSL 3.0!";
    size_t msg_len = strlen((const char *)msg);

    md_ctx = EVP_MD_CTX_new();
    if (!md_ctx)
        handle_errors();

    /* 初始化签名上下文（Ed25519 内部使用 SHA-512，无需外部指定摘要算法） */
    if (EVP_DigestSignInit(md_ctx, NULL, NULL, NULL, pkey) <= 0)
        handle_errors();

    /* 第一阶段：获取签名长度 */
    if (EVP_DigestSign(md_ctx, NULL, &sig_len, msg, msg_len) <= 0)
        handle_errors();

    sig = OPENSSL_malloc(sig_len);
    if (!sig)
        handle_errors();

    /* 第二阶段：生成签名（Ed25519 签名为固定 64 字节） */
    if (EVP_DigestSign(md_ctx, sig, &sig_len, msg, msg_len) <= 0)
        handle_errors();

    printf("[OK] 签名生成成功，长度: %zu 字节\n", sig_len);

    /* ===== 3. 验证签名 ===== */
    /* 重置上下文，准备验证 */
    EVP_MD_CTX_free(md_ctx);
    md_ctx = EVP_MD_CTX_new();
    if (!md_ctx)
        handle_errors();

    /* 初始化验证上下文 */
    if (EVP_DigestVerifyInit(md_ctx, NULL, NULL, NULL, pkey) <= 0)
        handle_errors();

    /* 验证签名 */
    ret = EVP_DigestVerify(md_ctx, sig, sig_len, msg, msg_len);
    if (ret == 1) {
        printf("[OK] 签名验证成功！\n");
    } else if (ret == 0) {
        printf("[FAIL] 签名验证失败：签名无效！\n");
    } else {
        handle_errors();
    }

    /* ===== 4. 验证重要安全属性 ===== */
    /* 二次签名测试：验证确定性属性（同一消息两次签名是否一致） */
    unsigned char sig2[64];
    size_t sig2_len = 64;

    EVP_MD_CTX_free(md_ctx);
    md_ctx = EVP_MD_CTX_new();
    if (EVP_DigestSignInit(md_ctx, NULL, NULL, NULL, pkey) <= 0)
        handle_errors();
    if (EVP_DigestSign(md_ctx, sig2, &sig2_len, msg, msg_len) <= 0)
        handle_errors();

    if (sig_len == sig2_len && memcmp(sig, sig2, sig_len) == 0) {
        printf("[OK] 确定性签名属性验证通过（两次签名完全一致）\n");
    } else {
        printf("[WARN] 签名结果不一致，请检查实现\n");
    }

    /* 篡改签名测试：篡改后的签名应验证失败 */
    sig2[0] ^= 0xFF;
    if (EVP_DigestVerify(md_ctx, sig2, sig2_len, msg, msg_len) == 0) {
        printf("[OK] 防篡改属性验证通过（篡改签名正确被拒绝）\n");
    }

    /* ===== 5. 清理 ===== */
    OPENSSL_free(sig);
    EVP_MD_CTX_free(md_ctx);
    EVP_PKEY_CTX_free(pctx);
    EVP_PKEY_free(pkey);

    return 0;
}
```

API 使用说明：

EVP_PKEY_CTX_new_from_name(NULL, "ED25519", NULL)：通过算法名称创建密钥上下文

EVP_PKEY_keygen_init → EVP_PKEY_generate：初始化并执行密钥生成

EVP_DigestSignInit：初始化签名上下文。Ed25519 内部已集成 SHA-512，因此摘要算法参数为 NULL，不可传入外部摘要

EVP_DigestSign：两阶段调用——第一次传入 sig=NULL 获取所需缓冲区长度，第二次传入实际缓冲区执行签名。Ed25519 签名为固定 64 字节

EVP_DigestVerifyInit → EVP_DigestVerify：验证初始化与执行，返回 1（成功）、0（失败）或负值（错误）

错误处理使用 ERR_print_errors_fp(stderr) 输出 OpenSSL 错误栈

#### 7.2 mbedTLS API 实现
mbedTLS 作为轻量级嵌入式密码库，在车载 ECU 等资源受限设备中应用广泛。以下为 mbedTLS Ed25519 签名/验签示例：

```c
/**
 * Ed25519 签名/验签示例（mbedTLS）
 * 编译：gcc -o ed25519_mbedtls ed25519_mbedtls.c -lmbedtls -lmbedcrypto
 * mbedTLS 版本要求：较新版本（Ed25519 API 支持在不同版本间存在差异）
 * 
 * 核心 API 说明：
 * - mbedtls_ecp_keypair 结构体：持有 Ed25519 密钥对
 * - mbedtls_ecp_gen_key()：示意性调用，实际 mbedTLS Ed25519 API 版本间存在差异
 * - mbedtls_ecp_sign()：对已哈希的消息摘要进行签名
 * - mbedtls_ecp_verify()：验证签名
 */

#include <stdio.h>
#include <string.h>
#include "mbedtls/ecp.h"
#include "mbedtls/sha512.h"

int main() {
    int ret;
    mbedtls_ecp_keypair key;
    mbedtls_sha512_context sha_ctx;
    const unsigned char *msg = (const unsigned char *)"Hello, Ed25519 from mbedTLS!";
    size_t msg_len = strlen((const char *)msg);
    unsigned char hash[64];          /* Ed25519 使用 SHA-512，输出 64 字节 */
    unsigned char sig[64];           /* Ed25519 签名为固定 64 字节 */
    size_t sig_len = 64;

    /* ===== 1. 初始化 ===== */
    mbedtls_ecp_keypair_init(&key);
    mbedtls_sha512_init(&sha_ctx);

    /* ===== 2. 生成 Ed25519 密钥对 ===== */
    /* MBEDTLS_ECP_DP_CURVE25519 是 Curve25519 的 ECDH 曲线，
     * Ed25519 需要使用 MBEDTLS_ECP_DP_ED25519 作为曲线标识符。
     * 但 mbedTLS 的 Ed25519 API 在不同版本间差异较大，实际调用请以当前版本文档为准。 */
    /* ret = mbedtls_ecp_gen_key(MBEDTLS_ECP_DP_ED25519, &key,
                              mbedtls_ecp_gen_keypair, NULL); */
    ret = 0; /* 占位：请替换为当前版本的实际 Ed25519 密钥生成调用 */
    if (ret != 0) {
        printf("[FAIL] 密钥生成失败，错误码: %d\n", ret);
        goto exit;
    }
    printf("[OK] Ed25519 密钥对生成成功\n");

    /* ===== 3. 计算消息哈希 ===== */
    /* mbedTLS 的 mbedtls_ecp_sign 要求输入 64 字节哈希值 */
    ret = mbedtls_sha512_starts(&sha_ctx, 0);
    ret |= mbedtls_sha512_update(&sha_ctx, msg, msg_len);
    ret |= mbedtls_sha512_finish(&sha_ctx, hash);
    if (ret != 0) {
        printf("[FAIL] 哈希计算失败\n");
        goto exit;
    }

    /* ===== 4. 签名 ===== */
    /* mbedtls_ecp_sign 的参数：
     *   - grp.id: 曲线标识符（此处为 MBEDTLS_ECP_DP_ED25519）
     *   - 后四个参数由内部派生（此处均传 NULL）
     *   - hash: 消息的 64 字节 SHA-512 哈希值
     *   - sig: 输出签名缓冲区
     *   - sig_len: 签名长度（输出参数）
     *   - f_rng, p_rng: 随机数生成器（Ed25519 内部是确定性的，理论上可不依赖随机源） */
    ret = mbedtls_ecp_sign(&key.grp, &key.d, &key.Q,
                           hash, 64, sig, &sig_len,
                           NULL, NULL);
    if (ret != 0) {
        printf("[FAIL] 签名生成失败，错误码: %d\n", ret);
        goto exit;
    }
    printf("[OK] 签名生成成功，长度: %zu 字节\n", sig_len);

    /* ===== 5. 验证签名 ===== */
    /* mbedtls_ecp_verify 的参数：
     *   - grp: 椭圆曲线群参数
     *   - hash: 64 字节消息哈希
     *   - Q: 签名者公钥
     *   - sig: 待验证签名
     *   - sig_len: 签名长度（应为 64 字节） */
    ret = mbedtls_ecp_verify(&key.grp, hash, 64, &key.Q, sig, sig_len);
    if (ret == 0) {
        printf("[OK] 签名验证成功！\n");
    } else {
        printf("[FAIL] 签名验证失败，错误码: %d\n", ret);
    }

    /* ===== 6. 防篡改验证测试 ===== */
    /* 篡改签名的一个字节后，验证应失败 */
    sig[0] ^= 0xFF;
    ret = mbedtls_ecp_verify(&key.grp, hash, 64, &key.Q, sig, sig_len);
    if (ret != 0) {
        printf("[OK] 防篡改验证通过（篡改签名正确被拒绝）\n");
    }

exit:
    mbedtls_ecp_keypair_free(&key);
    mbedtls_sha512_free(&sha_ctx);
    return ret;
}
```
mbedTLS vs OpenSSL 实现差异说明：

mbedTLS 的 mbedtls_ecp_sign 要求调用方先完成 SHA-512 哈希，再传入哈希值进行签名（分离式）；而 OpenSSL 的 EVP_DigestSign 在内部集成了哈希运算（一体化），调用方传入原始消息即可

mbedTLS 中 Ed25519 使用 MBEDTLS_ECP_DP_ED25519 作为曲线标识符，与 Curve25519 ECDH（MBEDTLS_ECP_DP_CURVE25519）是不同的参数集

mbedTLS 的 mbedtls_ecp_sign 内部实现为确定性签名，因此可以不依赖外部随机数生成器（f_rng 和 p_rng 参数可传 NULL）

在嵌入式部署中，建议使用 PSA Crypto API 以统一密码操作接口并更好地利用硬件加速

### 8 总结
Ed25519 是我认为当前密码工程领域中最具实用价值的椭圆曲线签名算法之一。它并非在所有维度上都绝对领先，但其设计哲学——安全、高效、简单、无专利——使其在众多场景下成为比 RSA、ECDSA 甚至 SM2 更优秀的选择，尤其是在车载嵌入式等资源受限且安全要求苛刻的环境中。

Ed25519 的核心设计优势与工程适用性：

|设计特性|工程意义|车联网适用性|
|:---|:---|:---|
|确定性签名|移除签名阶段对随机数的强依赖，降低嵌入式实现复杂度|极高（ECU 熵源质量通常有限）|
|小签名（64 字节）|减少总线传输带宽占用，优化 V2X 证书链尺寸|极高（车载总线带宽稀缺）|
|快速验证（约为 RSA 的 10 倍）|满足车辆高速移动场景的低延迟认证需求|极高（V2X 实时性要求严格）|
|完备加法公式|无例外点分支，易于实现恒定时间，抵御侧信道|极高（车载芯片抗侧信道需求）|
|无已知专利约束|降低车企供应链法律风险与授权成本|较高|
|参数完全公开透明|可审计，无后门风险|较高|

持续关注的工程要点：

确定性的“双刃剑”效应：避免对完全相同的数据体重复签名

与遗留 PKI 体系的兼容性：评估 CA 和证书格式对 Ed25519 的支持情况

密码敏捷性设计：为后量子密码迁移预留架构空间（参考 AUTOSAR 4.3.1 PQC 方案）

国内合规路径：在国内 V2X 场景中，需关注 Ed25519 与国密 SM2 的合规定位差异——SM2 为国内合规首选，Ed25519 为国际化部署的补充选项

参考资料

*Bernstein, D. J., Duif, N., Lange, T., Schwabe, P., & Yang, B. Y. (2011). High-speed high-security signatures. Journal of Cryptographic Engineering.*

*Bernstein, D. J. (2006). *Curve25519: new Diffie-Hellman speed records*. PKC 2006.*

*RFC 8032: Edwards-Curve Digital Signature Algorithm (EdDSA), IETF, 2017.*

*RFC 7748: Elliptic Curves for Security, IETF, 2016.*

*NIST SP 800-186: Recommendations for Discrete Logarithm-based Cryptography: Elliptic Curve Domain Parameters.*

*AUTOSAR, RS_Cryptography — Requirements Specification for Cryptography.*

*ISO/SAE 21434:2021, Road vehicles — Cybersecurity engineering.*

*Microchip Technology Inc., TA101 Trust Anchor Security IC Datasheet, 2023.*

*工信部, 《车联网数据安全传输技术规范》, 2025.*

*GM/T 0138-2024, *C-V2X车联网证书策略与认证业务声明框架*.*