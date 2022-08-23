---
layout: post
title: 盲签名算法的原理与C语言实现
time: 2022年6月15日
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: cryptography
tags: [cryptography]
excerpt_separator:  <!--more-->
---

## 0x01 概述

**盲签名(Blind Signature)** 是由Chaum,David提出的一种数字签名方式，其中消息的内容在签名之前对签名者是不可见的（盲化）。经过盲签名得到的签名值可以使用原始的非盲消息使用常规数字签名验证的方式进行公开验证。盲签名可以有效的保护隐私，其中签名者和消息作者不同，在电子投票系统和数字现金系统中会被使用。

盲签名常常被类比成下面的场景：Alice想让Bob在自己的文件上签名，但是不希望Bob看到文件内容，于是Alice在文件上方叠放了一张复写纸，然后将文件和复写纸放入信封密封起来交给Bob。Bob再拿到信封后验证了Alice的身份后，直接在密封好的信封上签字，这样虽然Bob是对密封后的信封签字，但是Alice拿到签名后的信封后，拆开信封就可以拿到经过Bob签字的文件。

<!--more-->

## 0x02 RSA盲签名方案

盲签名是一种消息在签名之前就被盲化处理的数字签名方案，盲签名可以使用很多公钥加密方案来实现。这里只介绍最简单的一种实现，基于RSA加密算法的盲签名方案。假设消息的持有者Alice希望对消息$m$使用盲签名方案进行签名，Bob是签名私钥的控制者，他们两方应该执行以下步骤：

- Alice选择一个随机数$k$作为盲化因子
- Alice对原始的消息进行计算，$m' = m k^e (mod \ n)$ 并把计算后（盲化）的消息 $m'$发送给Bob
- Bob计算 $s' = (m')^d (mod \ n)$ 并把计算后的签名值 $s'$ 发送给Alice
- Alice计算 $s = s'k^{-1} (mod \ n)$，$s$ 就是Bob对原始消息 $m$的数字签名

证明：

$(m')^d \equiv (m k^e)^d \equiv m^d k \  (mod \ n) $

$(m')^d k^{-1} = (m k^e)^d k^{-1} = m^d k^{e d} k^{-1} = m^d k k^{-1} \equiv m^d$

## 0x03 C语言实现

因为需要使用大数运算，可以使用你熟悉的任何语言实现，也可以用任意成熟的大数运算库实现。这里我使用了mbedTLS的大数运算库。

```c
//首先使用盲化银子blind_factor对原始的消息m进行盲化，生成盲化消息blind_message
int blindsignature_hide_message(mbedtls_mpi* m, mbedtls_mpi* blind_factor, mbedtls_mpi* e, mbedtls_mpi* n, mbedtls_mpi* blind_message)
{
	int ret;
	mbedtls_mpi r;
	mbedtls_mpi_init(&r);

	// r = blind_factor ^ e mod n
	if ((ret = mbedtls_mpi_exp_mod(&r, blind_factor, e, n, NULL) ) != 0)
	{
		printf("Hide message: mbedtls_mpi_mod_init failed ret=%8X\r\n", ret);
		goto EXIT;
	}
	// m1 = m * r
	if ((ret = mbedtls_mpi_mul_mpi(blind_message, m, &r)) != 0)
	{
		printf("Hide message: mbedtls_mpi_mul_mpi failed ret=%08X\r\n", ret);
		goto EXIT;
	}
	// blind_message = m1 mod n
	if ((ret = mbedtls_mpi_mod_mpi(blind_message, blind_message, n)) != 0)
	{
		printf("Hide message: mbedtls_mpi_mod_mpi failed ret=%08X\r\n", ret);
		goto EXIT;
	}
EXIT:
	mbedtls_mpi_free(&r);
	return 0;
}

//对盲化的消息进行盲签名，过程同普通rsa签名一样
int blindsignature_sign(mbedtls_mpi* blind_message, mbedtls_mpi* d, mbedtls_mpi* n, mbedtls_mpi* s)
{
	int ret;
	mbedtls_mpi r;

	//s = m ^d mod n
	if ((ret = mbedtls_mpi_exp_mod(s, blind_message, d, n, NULL)) != 0)
	{
		printf("Blind signature: mbedtls_mpi_exp_mod failed ret=%08X\r\n", ret);
		goto EXIT;
	}
EXIT:
	return 0;
}

//对签名结果blind_signature，使用blind_factor去盲化，得到签名值signature
int blindsignature_unblind_sign(mbedtls_mpi* blind_signature, mbedtls_mpi* blind_factor, mbedtls_mpi* n, mbedtls_mpi* signature)
{
	int ret;
	mbedtls_mpi inv_blind_factor;

	mbedtls_mpi_init(&inv_blind_factor);

	if ((ret = mbedtls_mpi_inv_mod(&inv_blind_factor, blind_factor, n)) != 0)
	{
		printf("Unblind signature: mbedtls_mpi_inv_mod failed ret=%08X\r\n", ret);
		goto EXIT;
	}

	if ((ret = mbedtls_mpi_mul_mpi(signature, &inv_blind_factor, blind_signature)) != 0)
	{
		printf("Unblind signature: mbedtls_mpi_mul_mpi failed ret=%08X\r\n", ret);
		goto EXIT;
	}

	if ((ret = mbedtls_mpi_mod_mpi(signature, signature, n)) != 0)
	{
		printf("Unblind signature: mbedtls_mpi_mod_mpi failed ret=%08X\r\n", ret);
		goto EXIT;
	}
EXIT:
	mbedtls_mpi_free(&inv_blind_factor);
	return 0;
}
```

## 0x04 总结

RSA盲签名的代码、调用示例程序都已经上传，可以直接检出测试，没有仔细测试验证可能有各种问题，如果遇到问题，可以给我提[issues](https://github.com/youngbug/blindsignatures_rsa/issues)。

```bash
git clone https://github.com/youngbug/blindsignatures_rsa.git
```