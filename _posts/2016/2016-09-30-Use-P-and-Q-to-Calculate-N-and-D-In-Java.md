---
layout: post
title: JAVA中使用P和Q分量计算N和D进行RSA运算
time: 2016年09月30日 星期五
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: Cryptography
tags: [cryptography,java]
excerpt_separator: <!--more-->
---

最近在使用Java中需要使用PQ形式的私钥进行RSA加解密运算，本来以为Java中应该很多类似的例子，发现所有的例子都是从ND形式的私钥，竟然没有人用分量P和Q计算N和D进行运算。对Java使用RSA运算不太熟，只能自己一点一点搞了。身边的Java 的仙们，好像身边都没人中国剩余定理，所以也不会遇到P和Q？不管他们了，开工了。

<!--more-->

## 1.BigInteger类
Java中有现成的大数运算的BigInteger类，直接使用这个类进行运算即可，总结一下使用中遇到的坑。Java的大数多1bit表示符号，所以如果1024byte的N在BigInteger中是1025bit，最高位多了1bit符号位，所以如果用BigInteger中的toByteArray()可以获得大数的二进制补码，如果需要导出BigInteger中的数据，需要忽略符号位，从第二字节开始拷贝，如果从第一字节就拷贝，那么会丢失最后一字节，把符号位存下来。
BigInteger类提供modInverse方法，可以直接求$d=e^{-1} = mod \\phi(n)$，这样就省事多了。

## 2.Cipher类
javax.crypto.Cipher类有个getInstance()方法，参数是“算法/模式/填充方式”，因为我只有一块定长128字节数据进行RSA运算，自己进行填充和去填充,按照sun的文档中的说明，填写"RSA/None/NoPadding"，但是编译的时候报错，提示不支持，网上搜了搜，都说默认的Crypt Provider不支持NoPadding，必须是PKCS#1的填充，感觉很不靠谱啊，后来发现是在Jdk1.7还是哪个版本之后，不支持None的模式，用ECB模式就行了，之前的版本是不是支持None也没去验证。

## 3.代码
**RSACrtUtil.java**
```java
package com.zhantianzuo.alg;

import java.math.BigInteger;
import java.security.Key;
import java.security.KeyFactory;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.NoSuchAlgorithmException;
import java.security.Security;
import java.security.interfaces.RSAPrivateCrtKey;
import java.security.interfaces.RSAPublicKey;
import java.security.spec.RSAPrivateKeySpec;
import java.security.spec.RSAPublicKeySpec;

import javax.crypto.Cipher;

/**
 * 
 * RSACrtUtil	RSA加解密 
 * 使用中国剩余定理类型的密钥
 * 私钥是P和Q，公钥是N,E固定0x10001
 *
 * @Date 2016.9.6
 *
 * @version v1.0
 *
 * @author 赵洋 cnrgc@163.com
 */

public class RSACrtUtil {
	
	public static final int RSA_MODULUS_LEN = 128;
	public static final int RSA_P_LEN = RSA_MODULUS_LEN/2;
	public static final int RSA_Q_LEN = RSA_MODULUS_LEN/2;
	public static final int publicExponent = 65537;
	public static final String KEY_ALGORITHM_MODE_PADDING = "RSA/ECB/NoPadding"; //不填充
	public static final String KEY_ALGORITHM = "RSA"; //不填充
	
	/**
	 * prikey_crt_decrypt 使用PQ的RSA私钥解密
	 * 私钥格式前半部分是P,后半部分是Q
	 * 
	 * */
	public static byte[] prikey_crt_decrypt(byte[] data, byte[] prikey) throws Exception{
		
		
		byte[] buf_p = new byte[RSA_P_LEN];
		byte[] buf_q = new byte[RSA_Q_LEN];
		//buf_p[0] = (byte)0x00;
		//buf_q[0] = (byte)0x00;
		System.arraycopy(prikey, 0, buf_p, 0, RSA_P_LEN);
		System.arraycopy(prikey, RSA_P_LEN, buf_q, 0, RSA_Q_LEN);
		//
		/**
		 *  1.p,q计算n
		 * */
		BigInteger p = new BigInteger(1, buf_p);
		BigInteger q = new BigInteger(1, buf_q);
		BigInteger n = p.multiply(q); //n = p * q
		/**
		 * 	2. 计算d = (p-1) * (q-1) mod e
		 * */
		BigInteger p1 = p.subtract(BigInteger.valueOf(1));
		BigInteger q1 = q.subtract(BigInteger.valueOf(1));
		BigInteger h = p1.multiply(q1);// h = (p-1) * (q-1)
		BigInteger e = BigInteger.valueOf(publicExponent);
		//BigInteger d = h.mod(e);
		BigInteger d = e.modInverse(h);

		/**
		 * 	3. 创建 RSA私钥
		 * */
		RSAPrivateKeySpec keyspec = new RSAPrivateKeySpec(n, d);
		KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);   
        Key privateKey = keyFactory.generatePrivate(keyspec);
		/**
		 * 	4. 数据解密 
		 * */
        Cipher cipher = Cipher.getInstance(KEY_ALGORITHM_MODE_PADDING);   
        cipher.init(Cipher.DECRYPT_MODE, privateKey);   
        /**
         * 	5. 返回结果
         * */
        return cipher.doFinal(data);  
	}
	/**
	 * pubkey_encrypt 公钥加密
	 * 密钥是N
	 * */
	public static byte[] pubkey_encrypt(byte[] data, byte[] pubkey) throws Exception{
		
		/**
		 *  1.初始化大数模n和公钥指数e
		 * */
		byte[] pubkey_buf = new byte[RSA_MODULUS_LEN+1];//多一字节符号位
		pubkey_buf[0] = (byte)0x00;
		System.arraycopy(pubkey, 0, pubkey_buf, 1, RSA_MODULUS_LEN);
		//
		BigInteger e = BigInteger.valueOf(publicExponent);
		BigInteger n = new BigInteger(pubkey_buf);
		/**
		 *  2.创建RSA公钥
		 * */
		//
		RSAPublicKeySpec keyspec = new RSAPublicKeySpec(n, e);
		KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);   
        Key publicKey = keyFactory.generatePublic(keyspec);
        /**
         *  3.数据加密
         * */
        Cipher cipher = Cipher.getInstance(KEY_ALGORITHM_MODE_PADDING);
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);   
        /**
         * 	5. 返回结果
         * */
        return cipher.doFinal(data);
	}
	
	public static void generateKeyPair(byte[] pubkey, byte[] prikey) throws Exception{
		
		KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance(KEY_ALGORITHM);
		keyPairGenerator.initialize(RSA_MODULUS_LEN*8);
		
		KeyPair keyPair = keyPairGenerator.generateKeyPair();
		RSAPublicKey publicKey = (RSAPublicKey)keyPair.getPublic();
		RSAPrivateCrtKey privateKey = (RSAPrivateCrtKey)keyPair.getPrivate();
		//
		BigInteger n = publicKey.getModulus();
		BigInteger p = privateKey.getPrimeP();
		BigInteger q = privateKey.getPrimeQ();
		/**
		 *  BigInteger 里有一个bit的符号位,所以直接用toByteArray会包含符号位,
		 *  在c的代码里没符号位,所以1024bit的n,java里BigInteger是1025bit长
		 *  直接拷贝128byte出来,正数第一个字节是是0,后面会丢掉最后一字节
		 * */
		System.arraycopy(n.toByteArray(), 1, pubkey, 0, 128);
		System.arraycopy(p.toByteArray(), 1, prikey, 0, 64);
		System.arraycopy(q.toByteArray(), 1, prikey, 64, 64);
		//
	}

}
```

**Test.java**
```java
package com.zhantianzuo.alg;

import java.math.BigInteger;
import java.security.Key;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.NoSuchAlgorithmException;
import java.security.interfaces.RSAPrivateCrtKey;
import java.security.interfaces.RSAPublicKey;


public class Test {

	private static final String ALGORITHM = "RSA";
	private static final int key_len = 128;
	
	public static void main(String[] args) {
		//
		byte[] pubkey = new byte[128];
		byte[] prikey = new byte[128];
		try {
			RSACrtUtil.generateKeyPair(pubkey, prikey);
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		//
		int i;
		byte[] plaintext = null;
		byte[] ciphertext = null;
		byte[] data = new byte[key_len];
		for(i=0; i<key_len; i++){
			
			data[i] = (byte)i;
		}
		//
		try {
			ciphertext = RSACrtUtil.pubkey_encrypt(data, pubkey);
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		//
		
		try {
			plaintext = RSACrtUtil.prikey_crt_decrypt(ciphertext, prikey);
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
	}

}
```
