---
layout: post
title: 中国剩余定理CRT、高斯算法和RSA低加密指数广播攻击
time: 2022年3月10日 星期四
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: Cryptography
tags: [cryptography, algorithm]
excerpt_separator:  <!--more-->
---

这篇讨论一下中国剩余定理(Chinese Remainder Theorem),高斯算法(Gauss's algorithm)解决同步线性同余(simultaneous linear congruences)的问题、简单的方法去解决小模数(small moduli)同余、RSA低加密指数广播攻击的原理(theorem to break the RSA algorithm when someone sends the same encrypted message to three recipients using the same exponent of e=3，又叫Johan Hastad广播攻击)

 <!--more-->

## 中国剩余定理 The Chinese Theorem

>定理：有整数$n_1,n_2,\cdots,n_r$，$gcd(n_i,n_j)=1，且 i\neq j$，那么线性同余系统
>
>$x\equiv c_1 (mod \ n_1)$
>
>$x\equiv c_2 (mod \ n_2)$
>
>$x\equiv c_3 (mod \ n_3)$
>
> $\cdots$
>
> $x\equiv c_r (mod \ n_r)$
>
>

定理说有唯一解，并不是说如何去求解。这个通常使用高斯算法(Gauss's algorithm)。中国剩余定理更多的时候是用在对RSA算法进行提速。

中国剩余定理在《孙子算经》中的问题是：今有物不知其数，三三数之剩二，五五数之剩三，七七数之剩二，问物几何？在现代数论种我们把它写成解同余问题。

>$x \equiv 2 (mod \ 3)$
>
>$x \equiv 3 (mod \ 5)$
>
>$x \equiv 2 (mod \ 7)$
>

## 高斯算法 Gauss's algorithm

>算法：有$N=n_1 n_2 \cdots n_r$那么
>
>$x \equiv c_1 N_1 d_1 + c_2 N_2 d_2 + \cdots + c_r N_r dr (mod \ N)$
>
>$N_i = N/n_i 和 d_i \equiv N_i^{-1}(mod \ n_i)$

### 《孙子算经》的例子

《孙子算经》上面原始的中国剩余定理的题目有：
>$n_1=3,n_2=5,n_3=7$
>
>$N=n_1n_2n_3 = 3 \times 5 \times 7 = 105$
>
>$c_1 = 2, c_2 = 3, c_3 = 2$

$N_1 = N/n_1 = 105 \div 3 = 35 \ 所以 d_1=35^{-1} (mod \ 3) = 2$

$N_2 = N/n_2 = 105 \div 5 = 21 \ 所以 d_2= 21……{-1}(mod \ 5) = 1$

$N_3 = N/n_3 = 105 \div 7 = 15 \ 所以 d_3= 15^{-1}(mod \ 7) = 1 \ 因此$ 

> $x=(2 \times 35 \times 2)+(3 \times 21 \times 1)+(2 \times 15 \times 1)= 233 \equiv 23 (mod \ 105)$


## 低加密指数广播攻击RSA Johan Hastad attack


Alice发送您相同的RSA加密消息m给三个接收方，使用了不同的模数$n_1,n_2,n_3$，这些模数互质，但是他们使用了相同的指数$e=3$。Eve恢复出了密文值$c_1,c_2,c_3$并且知道三个接收方的公钥$(n, e=3)$。Eve是否可以在不分解模数的情况下，恢复出消息？

可以。Eve使用高斯算法可以找到解x，在$0 \le x \lt n_1 n_2 n_3 $范围内，
>$x \equiv c_1 (mod \ n1)$
>
>$x \equiv c_2 (mod \ n2)$
>
>$x \equiv c_3 (mod \ n3)$

我们知道$m^3 \lt n_1 n_2 n_3$，因此可以得到，$x=m^3$,$m$可以通过简单的对整数$x$求立方根恢复出来。

### 例子
有三个接收方的公钥$(87,3),(115,3)和(187,3)$，我们知道$e=3$并且
> $n_1=29 \times 3 = 87, n_2=23 \times 5= 115, n_3=17*11=187$
（实际使用中，会使用更大的N，不可以分解）

Alice使用RSA算法加密消息$m=10$给三个接收方，如下：
>$c_1=10^3 mod \ 87 = 43;c_2=10^3 mod \ 115=80;c_3= 10^3 mod \ 187=65 $

这三个密文值$c_1,c_2,c_3$被中间人Eve拦截，Eve知道公钥$(n_i, e)$。她可以使用高斯算法如下：
>$N=n_1 n_2 n_3 = 87 \times 115 \times 187 = 1870935$
>
>$N_1 = N/n_1 = 115 \times 187 = 21505; d_1= 20505^{-1}(mod \ 87) = 49$
>
>$N_2 = N/n_2 = 87 \times 187 = 16269; d_2 = 16269^{-1} (mod \ 115)=49$
>
>$N_3 = N/n_3 = 87 \times 115 = 10005; d_3 = 10005^{-1}(mod \ 187) = 2$
>
>$x \equiv c_1 N_1 d_1 + c_2 N_2 d_2 +c_3 N_3 d_3 (mod N)$
>
>$x = (43 \times 21505 \times 49) + (80 \times 16269 \times 49) + (65 \times 10005 \times 2) = 110386165 \equiv 1000 (mod \ 1870935)$

所以明文消息$m$是1000的立方根，$m=10$。所以Eve不需要对模数进行分解就可以恢复出明文消息。

## 如何防止以上的攻击
- 1. 使用大指数，比如65537(0x10001)。这样使用上面的攻击方法将会变得很困难。
- 2. 添加一些随机比特到消息中，至少64比特。确保每次消息加密都添加了不同的随机数。这种加盐的方法也可以防止许多其他的攻击。显然，接收方也需要知道如何在解密后去除填充的随机数。