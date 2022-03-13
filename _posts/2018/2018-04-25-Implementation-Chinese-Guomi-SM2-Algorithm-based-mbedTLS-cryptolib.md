---
layout: post
title: 基于mbedTLS算法库实现国密SM2签名和验签算法
author: Zhao Yang(cnrgc@163.com)
time: 2018年04月25日 星期三
location: 北京
pulished: true
category: Cryptography
tags: [cryptography,c]
excerpt_separator: <!--more-->
---

网上有大量的基于OpenSSL实现的国密算法库，比如著名的GmSSL,可以直接拿来用。我自己常用的是mbedTLS的算法库，比较小巧简单，在mbedTLS的大数算法的基础上实现了国密SM2的签名和验签算法。在基于mbedTLS实现SM2签名和验签算法的过程中走过一些弯路，现在把实现的过程记录下来备忘。

<!--more-->

国密SM2算法也是基于椭圆曲线公钥算法，椭圆曲线上的运算都是和国际算法一样的，国密SM2规范中给出了推荐曲线，所以首先需要加载国密推荐参数。

mbedTLS中使用ecp_group_load函数加载参数，需要定义一下SM2的椭圆曲线，在定义曲线参数时字节序跟SM2规范的上的顺序不一样，这里需要注意一下，当时在这里折腾了很久。

```c
static const mbedtls_mpi_uint sm2256_p[] = {
	BYTES_TO_T_UINT_8(0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF),
	BYTES_TO_T_UINT_8(0x00, 0x00, 0x00, 0x00, 0xFF, 0xFF, 0xFF, 0xFF),
	BYTES_TO_T_UINT_8(0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF),
	BYTES_TO_T_UINT_8(0xFF, 0xFF, 0xFF, 0xFF, 0xFE, 0xFF, 0xFF, 0xFF),
};
static const mbedtls_mpi_uint sm2256_a[] = {
	BYTES_TO_T_UINT_8(0xFC, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF),
	BYTES_TO_T_UINT_8(0x00, 0x00, 0x00, 0x00, 0xFF, 0xFF, 0xFF, 0xFF),
	BYTES_TO_T_UINT_8(0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF),
	BYTES_TO_T_UINT_8(0xFF, 0xFF, 0xFF, 0xFF, 0xFE, 0xFF, 0xFF, 0xFF),
};
static const mbedtls_mpi_uint sm2256_b[] = {
	BYTES_TO_T_UINT_8(0x93, 0x0E, 0x94, 0x4D, 0x41, 0xBD, 0xBC, 0xDD),
	BYTES_TO_T_UINT_8(0x92, 0x8F, 0xAB, 0x15, 0xF5, 0x89, 0x97, 0xF3),
	BYTES_TO_T_UINT_8(0xA7, 0x09, 0x65, 0xCF, 0x4B, 0x9E, 0x5A, 0x4D),
	BYTES_TO_T_UINT_8(0x34, 0x5E, 0x9F, 0x9D, 0x9E, 0xFA, 0xE9, 0x28),
};
static const mbedtls_mpi_uint sm2256_gx[] = {
	BYTES_TO_T_UINT_8(0xC7, 0x74, 0x4C, 0x33, 0x89, 0x45, 0x5A, 0x71),
	BYTES_TO_T_UINT_8(0xE1, 0x0B, 0x66, 0xF2, 0xBF, 0x0B, 0xE3, 0x8F),
	BYTES_TO_T_UINT_8(0x94, 0xC9, 0x39, 0x6A, 0x46, 0x04, 0x99, 0x5F),
	BYTES_TO_T_UINT_8(0x19, 0x81, 0x19, 0x1F, 0x2C, 0xAE, 0xC4, 0x32),
};
static const mbedtls_mpi_uint sm2256_gy[] = {
	BYTES_TO_T_UINT_8(0xA0, 0xF0, 0x39, 0x21, 0xE5, 0x32, 0xDF, 0x02),
	BYTES_TO_T_UINT_8(0x40, 0x47, 0x2A, 0xC6, 0x7C, 0x87, 0xA9, 0xD0),
	BYTES_TO_T_UINT_8(0x53, 0x21, 0x69, 0x6B, 0xE3, 0xCE, 0xBD, 0x59),
	BYTES_TO_T_UINT_8(0x9C, 0x77, 0xF6, 0xF4, 0xA2, 0x36, 0x37, 0xBC),
};
static const mbedtls_mpi_uint sm2256_n[] = {
	BYTES_TO_T_UINT_8(0x23, 0x41, 0xD5, 0x39, 0x09, 0xF4, 0xBB, 0x53),
	BYTES_TO_T_UINT_8(0x2B, 0x05, 0xC6, 0x21, 0x6B, 0xDF, 0x03, 0x72),
	BYTES_TO_T_UINT_8(0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF),
	BYTES_TO_T_UINT_8(0xFF, 0xFF, 0xFF, 0xFF, 0xFE, 0xFF, 0xFF, 0xFF),
};
```

使用这个曲线后，就可以尝试产生一下SM2密钥对了，生成之后可以用其他的支持SM2的算法工具或者算法库来验证，如果没问题，就可以进入下一步，实现SM2签名算法。

SM2的签名算法和ECC的签名过程是有区别的，SM2的过程是：

1.对待签名数据进行哈希算法（国密规范里还规定了使用用户ID，曲线参数等生成Z的过程，这里不考虑那些过程，直接处理最后哈希后的数据）

2.先生成一个SM2密钥对，私钥：k，公钥：kG = (x,y);

3.计算r = (e+x) mod n；

4.如果r=0 或者r+k=n返回步骤2；

5.s=（（1+d）^-1）(k-rd) mod n ;

6.如果s=0 返回 2;

7.签名结果（r,s）.

实现签名的代码如下：

```c
/**
* Compute ECDSA-SM2 signature of a hashed message
* Author: Zhao Yang cnrgc@163.com/sxzhaoyang@gmail.com
* Data: April 25 2018
*/
 
int mbedtls_ecdsa_sm2_sign(mbedtls_ecp_group *grp, mbedtls_mpi *r, mbedtls_mpi *s,
						const mbedtls_mpi *d, const unsigned char *buf, size_t blen,
						int(*f_rng)(void *, unsigned char *, size_t), void *p_rng)
{
	int ret, key_tries, sign_tries, blind_tries;
	mbedtls_ecp_point R;
	mbedtls_mpi  k, e, t, l, m;
	/* Fail cleanly on curves such as Curve25519 that can't be used for ECDSA */
	if (grp->N.p == NULL)
		return(MBEDTLS_ERR_ECP_BAD_INPUT_DATA);
 
	mbedtls_ecp_point_init(&R);
	mbedtls_mpi_init(&k); mbedtls_mpi_init(&e); mbedtls_mpi_init(&t); mbedtls_mpi_init(&l);
	mbedtls_mpi_init(&m);
 
	sign_tries = 0;
	do
	{
		/*
		* Step 0: derive MPI from hashed message
		*/
		MBEDTLS_MPI_CHK(derive_mpi(grp, &e, buf, blen));
		/*
		*		Step 1-3:
		*		set r = (e+x) mod n
		*/
		key_tries = 0;
		do
		{
			MBEDTLS_MPI_CHK(mbedtls_ecp_gen_keypair(grp, &k, &R, f_rng, p_rng));
			MBEDTLS_MPI_CHK(mbedtls_mpi_add_mpi(&l, &e, &R.X));
			MBEDTLS_MPI_CHK(mbedtls_mpi_mod_mpi(r, &l, &grp->N));
			
			if (key_tries++ > 10)
			{
				ret = MBEDTLS_ERR_ECP_RANDOM_FAILED;
				goto cleanup;
			}
			//r+k != n
			MBEDTLS_MPI_CHK((mbedtls_mpi_add_mpi(&m, r, &k)));
		} while ((mbedtls_mpi_cmp_int(r, 0) == 0)|| (mbedtls_mpi_cmp_mpi(&m, &grp->N) == 0));
		/*
		* Generate a random value to blind inv_mod in next step,
		* avoiding a potential timing leak.
		*/
		blind_tries = 0;
		do
		{
			size_t n_size = (grp->nbits + 7) / 8;
			MBEDTLS_MPI_CHK(mbedtls_mpi_fill_random(&t, n_size, f_rng, p_rng));
			MBEDTLS_MPI_CHK(mbedtls_mpi_shift_r(&t, 8 * n_size - grp->nbits));
 
			/* See mbedtls_ecp_gen_keypair() */
			if (++blind_tries > 30)
				return(MBEDTLS_ERR_ECP_RANDOM_FAILED);
		} while (mbedtls_mpi_cmp_int(&t, 1) < 0 ||
			mbedtls_mpi_cmp_mpi(&t, &grp->N) >= 0);
 
		/*
		* Step 6: compute  s = ((1+d)^-1)*(k-r*d) mod n
		* 
		*/
		MBEDTLS_MPI_CHK(mbedtls_mpi_mul_mpi(s, r, d)); //s = r*d
		MBEDTLS_MPI_CHK(mbedtls_mpi_sub_mpi(s, &k, s));   //s = k - s
		MBEDTLS_MPI_CHK(mbedtls_mpi_mul_mpi(s, s, &t));//s = s*t
		MBEDTLS_MPI_CHK(mbedtls_mpi_add_int(&l, d, 1));//l = 1+d
		MBEDTLS_MPI_CHK(mbedtls_mpi_mul_mpi(&l, &l, &t));//l=l*t
		MBEDTLS_MPI_CHK(mbedtls_mpi_inv_mod(&l, &l, &grp->N));// l = l^-1
		MBEDTLS_MPI_CHK(mbedtls_mpi_mul_mpi(s, s, &l));//s = s * l 
		MBEDTLS_MPI_CHK(mbedtls_mpi_mod_mpi(s, s, &grp->N));//s mod n
 
		if (sign_tries++ > 10)
		{
			ret = MBEDTLS_ERR_ECP_RANDOM_FAILED;
			goto cleanup;
		}
		//
	} while (mbedtls_mpi_cmp_int(&t, 1) < 0 ||
		     mbedtls_mpi_cmp_mpi(&t, &grp->N) >= 0);
cleanup:
	mbedtls_ecp_point_free(&R);
	mbedtls_mpi_free(&k); mbedtls_mpi_free(&e); mbedtls_mpi_free(&t);
	mbedtls_mpi_free(&l); mbedtls_mpi_free(&m);
	return (ret);
}

```
```c
/*
* Deterministic Guomi SM2 signature wrapper
* Author: Zhao Yang cnrgc@163.com/sxzhaoyang@gmail.com
* Data: April 25 2018
*/
int mbedtls_ecdsa_sm2_sign_det(mbedtls_ecp_group *grp, mbedtls_mpi *r, mbedtls_mpi *s,
	const mbedtls_mpi *d, const unsigned char *buf, size_t blen,
	mbedtls_md_type_t md_alg)
{
	int ret;
	mbedtls_hmac_drbg_context rng_ctx;
	unsigned char data[2 * MBEDTLS_ECP_MAX_BYTES];
	size_t grp_len = (grp->nbits + 7) / 8;
	const mbedtls_md_info_t *md_info;
	mbedtls_mpi h;
 
	if ((md_info = mbedtls_md_info_from_type(md_alg)) == NULL)
		return(MBEDTLS_ERR_ECP_BAD_INPUT_DATA);
 
	mbedtls_mpi_init(&h);
	mbedtls_hmac_drbg_init(&rng_ctx);
 
	/* Use private key and message hash (reduced) to initialize HMAC_DRBG */
	MBEDTLS_MPI_CHK(mbedtls_mpi_write_binary(d, data, grp_len));
	MBEDTLS_MPI_CHK(derive_mpi(grp, &h, buf, blen));
	MBEDTLS_MPI_CHK(mbedtls_mpi_write_binary(&h, data + grp_len, grp_len));
	mbedtls_hmac_drbg_seed_buf(&rng_ctx, md_info, data, 2 * grp_len);
 
	ret = mbedtls_ecdsa_sm2_sign(grp, r, s, d, buf, blen,
		mbedtls_hmac_drbg_random, &rng_ctx);
 
cleanup:
	mbedtls_hmac_drbg_free(&rng_ctx);
	mbedtls_mpi_free(&h);
 
	return(ret);
}
```

然后实现SM2的验证签名算法，同样SM2的验证过程跟ECC也有差别，验证过程如下：

1.e = hash(m);

2.计算t = (r + s) mod n，如果t=0验签失败;

3.计算椭圆曲线上的点(x,y) = sG + tP

4.计算R = (e + x) mod n 如果R=r那么签名正确，否则签名验证失败.

实现验证签名代码如下：

```c
/*
* Verify ECDSA Guomi SM2 signature of hashed message 
* Author: Zhao Yang cnrgc@163.com/sxzhaoyang@gmail.com
* Data: April 25 2018
*/
int mbedtls_ecdsa_sm2_verify(mbedtls_ecp_group *grp,
	const unsigned char *buf, size_t blen,
	const mbedtls_ecp_point *Q, const mbedtls_mpi *r, const mbedtls_mpi *s)
{
	int ret;
	mbedtls_mpi e, s_inv, u1, u2, t, result;
	mbedtls_ecp_point R;
 
	mbedtls_ecp_point_init(&R);
	mbedtls_mpi_init(&e); mbedtls_mpi_init(&s_inv); mbedtls_mpi_init(&u1); mbedtls_mpi_init(&u2);
	mbedtls_mpi_init(&t); mbedtls_mpi_init(&result); 
 
	/* Fail cleanly on curves such as Curve25519 that can't be used for ECDSA */
	if (grp->N.p == NULL)
		return(MBEDTLS_ERR_ECP_BAD_INPUT_DATA);
 
	/*
	* Step 1: make sure r and s are in range 1..n-1
	*/
	if (mbedtls_mpi_cmp_int(r, 1) < 0 || mbedtls_mpi_cmp_mpi(r, &grp->N) >= 0 ||
		mbedtls_mpi_cmp_int(s, 1) < 0 || mbedtls_mpi_cmp_mpi(s, &grp->N) >= 0)
	{
		ret = MBEDTLS_ERR_ECP_VERIFY_FAILED;
		goto cleanup;
	}
 
	/*
	* Additional precaution: make sure Q is valid
	*/
	MBEDTLS_MPI_CHK(mbedtls_ecp_check_pubkey(grp, Q));
 
	/*
	* Step 3: derive MPI from hashed message
	*/
	MBEDTLS_MPI_CHK(derive_mpi(grp, &e, buf, blen));
 
	/*
	* Step 4: t = (r+s) mod n
	*/
	MBEDTLS_MPI_CHK(mbedtls_mpi_add_mpi(&t, r, s));
	MBEDTLS_MPI_CHK(mbedtls_mpi_mod_mpi(&t, &t, &grp->N));
	if (mbedtls_mpi_cmp_int(&t, 0) == 0)
	{
		ret = MBEDTLS_ERR_ECP_VERIFY_FAILED;
		goto cleanup;
	}
	/*
	* Step 5: (x,y) = sG + tQ
	*/
	MBEDTLS_MPI_CHK(mbedtls_ecp_muladd(grp, &R, s, &grp->G, &t, Q));
	/*
	* Step 6: result = (e+x) mod n
	*/
	MBEDTLS_MPI_CHK(mbedtls_mpi_add_mpi(&e, &e, &R.X));
	MBEDTLS_MPI_CHK(mbedtls_mpi_mod_mpi(&result, &e, &grp->N));
	/*
	* Step 7: check if result.X (that is, result.X) is equal to r
	**/
	if (mbedtls_mpi_cmp_mpi(&result, r) != 0)
	{
		ret = MBEDTLS_ERR_ECP_VERIFY_FAILED;
		goto cleanup;
	}
	//
cleanup:
	mbedtls_ecp_point_free(&R); 
	mbedtls_mpi_free(&e); mbedtls_mpi_free(&s_inv); mbedtls_mpi_free(&u1); mbedtls_mpi_free(&u2);
	mbedtls_mpi_free(&t); mbedtls_mpi_free(&result);
	return(ret);
}
```
