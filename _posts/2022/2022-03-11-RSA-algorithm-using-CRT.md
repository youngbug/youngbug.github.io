---
layout: post_layout
title: 使用中国剩余定理CRT对RSA运算进行加速
time: 2022年3月10日 星期四
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
excerpt_separator: ""
---

这篇讲一下如何使用[中国剩余定理CRT](https://www.jianshu.com/p/8ebccf708c40)来对RSA加密运算进行加速。

## RSA运算
当我们使用RSA私钥(n,d)对密文c进行解密(或者计算数字签名时)，我们需要计算模幂$m=c^d mod \ n$。私钥指数$d$并不像公钥指数$e$那样方便。一个k比特的模n，对应的私钥指数d差不多跟它一样长。计算的工作量同长度k成正比，所以对于RSA私钥的运算，有更多的计算量。

我们可以使用CRT模式更有效的计算$m=c^d$

- 1. 使用$p,q，p \gt q$提前计算以下值:
>$dP = e^{-1} mod \ (p-1)$  
>$dQ=e^{-1} mod \ (q-1)$  
>$qInv = q^{-1} mod \ p$  

$e^{-1}$表示模逆，表达式$x=e^{-1} mod \ N$y也会写成$x=（1/e) mod \ N$。x是任意整数满足$x \cdot e \equiv 1 (mod \ N)$。$N=n=pq$。

- 2. 使用密文c计算明文消息m
>$m_1 = c^{dP} mod \ p$  
>$m_2 = c^{dQ} mod \ q$  
>$h = qInv \cdot (m_1-m_2) mod p$  
>$m = m_2 + h \cdot q$

我们把$(p,q,dP,dQ,qInv)$作为私钥保存。

下面需要了解两个数论的原理，分别是中国剩余定义的一个特殊情况和欧拉定理。

## 中国剩余定理-特殊情况
中国剩余定理的特殊情况可以表述如下：
>$p和q是不相同的素数，n=p \cdot q.对于任意的一对(x_1,x_2)，0 \leq x_1 \lt p 且 0 \leq x_2 \lt q，存在唯一的数x，0 \leq x \lt n  $  
>$x_1=x \ mod \ p, 且 x_2 =x \ mod \ q$  
所以任意整数x都可以使用CRT表示方法唯一的表示成$(x_1,x_2)$。

## 欧拉定理 Euler's Theorem
欧拉定理是费马小定理(Fermat's Little Theorem)的推广，也称作欧拉-费马定理(Euler-Fermat Theorem)。
> 如果n是一个正整数，a是任意整数，且$gcd(a,n)=1$，那么$a^{\phi(n)} \equiv 1 \ (mod \ n),\phi(n)是Euler's totient函数，求小于n的正整数中与n互质的个数$

一个质数p，$\phi(n)=p-1$

## CRT表示法中的运算
我们需要计算$m=c^d \ mod \ n$。如果我们知道$(c^d \ mod \ p, c^d \ mod q)$那么CRT告诉我们存在唯一的值$c^d  \ mod \ n$在范围[0,n-1]。

使用CRT表示方法$(x_1,x_2)$恢复出x，我们使用Garner's方程式。
> $x=x_2 + h \cdot q$  
>$h=((x_1-x_2)(q^{-1} \ mod \ p)) \ mod \ p$  
CRT系数$qInv = q^{-1} \ mod \ p$可以提前计算。模幂的运算量随着模的比特数k的立方增加而增加。所以做两次幂运算mod p和mod q，比做一次幂运算mod n效率要高。

计算$c^d \ mod \ p$，可以使用欧拉定理来减少指数d modulo (p-1):
> $c^d \ mod \ p = c^{d \ mod \ \phi(p)} \ mod \ p = c^{d \ mod \ (p-1)} \ mod \ p $    

对于q使用相同的算法。

## RSA运算

> $d \ mod \ (p-1)=e^{-1} \ mod \ (p-1),$
> $d \ mod \ (q-1) = e^{-1} \ mod \ (q-1).$  

> $dP = e^{-1} \ mod \ (p-1) = d \ mod \ (p-1)$
> $dQ = e^{-1} \ mod \ (q-1) = d \ mod \ (q-1)$
> $m_1 = c^{dP} \ mod \ p$
> $m_2 = c^{dQ} \ mod \ q$

> $qInv = q^{-1} \ mod \ p$
> $h=qInv \cdot (m_1-m_2) \ mod \ p$
> $m=m_2+h \cdot q$

