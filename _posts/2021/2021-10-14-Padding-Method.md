---
layout: post_layout
title: 加密算法中的填充方法
time: 2021年10月14日 星期四
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: Cryptography
tags: [cryptography, algorithm]
excerpt_separator: "#"
---
在对数据进行加解密,签名,计算MAC的时候,有时需要对数据进行填充,填充的方法主要分为两大类,一种是比特填充(Bit Padding),填充时以比特为最小单位进行填充,另一种是字节填充(Byte Padding),填充时以字节为最小单位进行填充.当然有时候我们按字节处理数据时,一些比特填充和字节填充的效果是一样,比如ISO/IEC 9797-1 Padding Method 2和ISO/IEC 7816-4,数据如果最小处理单位都是字节的话,填充的效果其实是一样的.
PKCS #1中定义的RSA算法用的Padding方法,这里先不说.

近期整理了一下加密算法中常用的填充方法,整理如下.

## 1.ISO/IEC 9797-1 Padding Method 1 (Zero Padding)
填充方法定义在ISO/IEC 9797-1标准MAC算法第二步填充方法中.

**ISO/IEC 9797-1 Padding Method 1**

>The data string D to be input to the MAC algorithm shall be right-padded with as few(possibly none)'0' bits as necessary to obtain a data string whose legth(in bits) is a positive integer multiple of n.

>NOTE1 MAC algorithms using Padding Method 1 may be subject to trivial forgery attacks.See informative Annex C for futher details.

>NOTE2 If the data string is empty,Padding Method 1 specifies that it is right-padded with n '0'bits.

填充方法为,在数据的右侧填充若干个'0'比特,一直填充到分组长度n,所以这个填充方法也叫做Zero Padding.如果数据长度等于分组长度,那么不需要填充.

eg.对10字节的数据data进行AES加密,填充到AES的分组长度128比特.
```
data: 00 01 02 03 04 05 06 07 08 09
```
在data右侧填充48比特'0'达到分组长度128比特
```
padding: 00 00 00 00 00 00
```
经过填充后data_padded: data||padding
```
data_padded: 00 01 02 03 04 05 06 07 08 09 00 00 00 00 00 00
```

## 2.ISO/IEC 9797-1 Padding Method 2 (ISO/IEC 7816-4)
填充方法定义在ISO/IEC 9797-1标准和ISO/IEC 7816-4中.

**ISO/IEC 9797-1 Padding Method 2**

>The data string D to be input to the MAC algorithm shall be right-padded with a single '1' bit. The resulting string shall then be right-padded with as few(possibly none)'0' bits as necessary to obtain a data string whose length(in bits) is a positive integer multiple of n.

>NOTE If the data string is empty,Padding Method 2 specifies that it is right-padded with a single '1' bit followed by n-1 '0' bits.

填充方法为,在数据的右侧先填充一个比特'1',剩下的部分填充比特'0',使得数据长度达到分组长度n.因为一般是按字节填充,其实就是先填充一个80,剩下的部分填充00.

eg.对10字节的数据data进行AES加密,填充到AES的分组长度128比特.
```
data: 00 01 02 03 04 05 06 07 08 09
```
在右侧先填充一个比特'1',剩下填充47比特的'0'
```
padding: 10000000B 00000000B 00000000B 00000000B 00000000B 00000000B
```
经过填充后
```
data_padded: 00 01 02 03 04 05 06 07 08 09 80 00 00 00 00 00
```

## 3.ISO/IEC 9797-1 Padding Method 3
填充方法定义在ISO/IEC 9797-1标准中.

**ISO/IEC 9797-1 Padding Method 3**
>The data string D to be input to the MAC algorithm shall be right-padded with as few(possibly none)'0' bits as necessary to obtain a data string whose length(in bits) is a positive integer multiple of n.The resulting string shall then be left-padded with a block L.The block L consists of the binary representation of the length(in bits)LD of the unpadded data string D, left-padded with as few(possibly none)'0' bits as necessary to obtain an n-bit block. The right-most bit of the block L corresponds to the least significant bit of the binary representation of LD.

>NOTE1 Padding Method 3 is not suitable for use in situations where the data string is not available prior to the start of the MAC calculation.

>NOTE2 If the data string is empty,Padding Method 3 specifies that it is right-padded with n '0' bits and left-padded with a block L consisting of n '0' bits.

应当在数据D的右侧填充填充多个'0'比特来获得一个新的数据,使得新数据长度为分组n的整数倍.新数据的左侧应填充一个数据块L.数据块L由数据D未填充前的长度(用比特表示)LD的二进制表示组成,左侧用多个'0'来填充来获得一个n比特的块.数据块L的最右一位对应LD二进制表示的最低有效位(LSB).

eg. 对10字节数据data使用Padding Method 3填充,到AES的分组16字节.
```
data: 00 01 02 03 04 05 06 07 08 09
```
```
padding1: 00 00 00 00 00 00
```
新数据有data右侧拼接padding1 data||padding1  
```
data_new: 00 01 02 03 04 05 06 07 08 09 00 00 00 00 00 00
```
数据块L由data未填充前的长度(比特)表示二进制组成,10字节是80(0x50)比特,左侧填充0,使得块L长度位AES的分组128bit
```
block L:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 50
```
最后填充后的结果是:L||data_new
```
data_padded: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 50 00 01 02 03 04 05 06 07 08 09 00 00 00 00 00 00
```
## 4.ISO/IEC 9797-1 Padding Method 4
填充方法定义在ISO/IEC 9797-1标准中.

**ISO/IEC 9797-1 Padding Method 4**

>If the data string D to be input to the MAC algorithm has a length(in bits) that is a postive integer multiple of n, no padding shall be applied.Otherwise, the data string D shall be right-padded with a single '1' bit. The resulting string shall then be right-padded with as few(possibly none)'0' as necessary to obtain a data string whose length(int bits)is a positive integer multiple of n.

>NOTE If the data string is empty, Padding Method 4 specifies that it is right-padded with a single '1' bit followed by n-1 '0' bits.

如果数据D的长度是分组长度n的整数倍,不需要添加填充.否则数据D的右侧需要填充一个比特'1',然后再填充多个比特'0'使得填充后的数据长度是分组n的整数倍.
Padding Method 4的填充方法和Padding Method 2的方法相似,区别是如果数据D的长度已经是分组n的整数倍,就不再填充了.

## 5.ANSI X9.23 Padding Method
ANSI X9.23是一个叫做"X9.23-1988 Financial Institution Encryption of Wholesale Financial Messages"的标准,这个标准已经被ASC X9 (Accredited Standards Committee X9) 撤销.一些应用可能还会依赖这个填充,所以还是会有一些X9.23填充的需求.在Java Bouncycastle中支持X923Padding,所以有的地方会有一个X.923的讹传的写法.

>The ANSI X9.23 method always appends from 1 - 8 bytes to the plaintext before encipherment. The last appended byte is the count of the added bytes and is in the range of X'01' - X'08'. The standard defines that any other added bytes, or pad characters, be random.

ANSI X9.23填充方法总是在加密之前向明文追加1~8个字节,最后一个添加的字节是填充字节的总数,范围是01~08.其他填充的字节是随机数.

在网上看到一些地方所谓的X923填充是最后一个字节是填充的字节个数,其他字节填充00,去填充的时候处理和上面其实是一样的,只是随机数变成固定的00了,安全性应该是下降了.

eg.对8字节数据data使用X9.23填充方法填充.
```
data: 01 02 03 04 05 06 07 08
```
填充1~8字节,因为本身数据长度是8字节,等于分组数据长度8,所以需要再填充一个8字节,最后一个字节填充08,剩下填充随机数.
```
padding: 04 AB 8B 4C 23 5D 33 08
```
填充后的数据为data||padding
```
data_padded: 01 02 03 04 05 06 07 08 04 AB 8B 4C 23 5D 33 08
```

## 6. ISO 10126 Padding(CBC mode only)
ISO 10126名称为Procedures for message encipherment(wholesale),规范的第六部分Padding field定义了填充方法.

>Padding shall be present in every CBC message.Plaintext shall be padded to a multiple of 64 bits before encipherment using the CBC mode.Padding shall be performed using either bits or octets by appending a padding field to the end of the plaintext.After decipherment,the padding field shall be discarded.
When padding with bits, the padding field shall consist of 8 to 71 bits divided into two subfields.The first subfield(pad fill) shall consist of 0 to 63 bits with arbitrary contents.The second sub-field(pad count) shall consist of 8 bits containing the number of bits in the padding field.In the range 8 to 71.The left-most bit of the pad count shall indicate that the pad count is given in bits(value of 1).The remaining bits of the pad count shall contain an unsigned binary number .That number shall be the total number of bits in the padding field.
When padding with octets,the padding field shall consist of 1 to 8 octets divided into two subfields.The first subfield(pad fill) shall consist of 0 to 7 octets with arbitary contents.The second subfield(pad count) shall consist of one octet containing the number of octets in the padding field in the range of 1 to 8.The left-most bit of the pad count shall indicate that the pad count is given in octets(value of 0).THe remaining bits of the pad count shall contain an unsigned binary number.That number shall be the total number of padding octets in the padding field.

每个CBC消息都需要填充,明文被填充为64比特的整数倍后再进行CBC模式加密.填充必须通过在明文末尾后追加比特或者字节(八位组)来实现.当解密后填充域将被丢弃.

当使用比特进行填充时,填充域由8~71比特组成,填充域被分为两个子域.第一个子域(填充pad)由0~63比特任意内容组成.第二个子域(pad个数)由8比特组成,其中包含填充域的比特个数.在8~71之间.第二个子域(pad个数域)的最左侧一个比特为'1'时指示为按比特填充.第二个子域(pad个数域)的其余比特应该包含一个无符号二进制数,这个数字表示填充域的比特总数.

当使用字节进行填充时,填充域应该由1~8个字节组成,填充域被分为两个子域.第一个子域(填充pad)由0~7字节组成,第一个子域包含任意内容.第二个子域(pad个数)由一个字节组成,这个字节包含了填充域中字节的数量,范围是1~8.第二个子域(pad个数域)最左边的一个比特为'0'时指示为按字节填充.第二个子域(pad个数域)的其余比特应该包含一个无符号的二进制数,这个数字表示填充域的比特总数.

eg.1 对5字节的数据使用ISO 10126 Padding按比特填充
```
data: 01 02 03 04 05
```
分组为64比特,数据为40比特,需要填充24特比特,第一个子域为16比特随机数,第二个子域8比特,最左侧的比特为1B,剩余部分为0000011B,第二个子域整体为10000011B.
```
padding_subfield1: AC 01
padding_subfield2: 83
```
填充域为 padding_subfield1||padding_subfield2
```
padding: AC 01 83
``` 
填充域追加在明文后面,填充后结果为 data||padding
```
data_padded: 01 02 03 04 05 AC 01 83
```
eg.2 对5字节的数据使用ISO 10126 Padding按字节填充
```
data: 01 02 03 04 05
```
分组为8字节,数据为5字节,需要填充3字节,第一个子域为2字节随机数,第二个子域为1字节,最左侧的比特位0B,剩余部分位3,第二个子域整体位03H.
```
padding_subfield1: 3C 01
padding_subfield2: 03
```
填充域为 padding_subfield1||padding_subfield2
```
padding: 3C 01 03
``` 
填充域追加在明文后面,填充后结果为 data||padding
```
data_padded: 01 02 03 04 05 3C 01 03
```
## 7.PKCS #5 Padding
PKCS #5中定义了的一种填充方法,属于字节填充,比较简单,填充1~8字节使得数据成为分组8的整数倍,填充的内容就是填充字节的个数.
>Concatenate M and a padding string PS to form an encoded message EM:
EM = M || PS
where the padding string PS consists of 8-(||M|| mod 8) octets each with value 8-(||M|| mod 8). The padding string PS will satisfy one of the following statements:
PS = 01, if ||M|| mod 8 = 7 ;
PS = 02 02, if ||M|| mod 8 = 6 ;
...
PS = 08 08 08 08 08 08 08 08, if ||M|| mod 8 = 0.
The length in octets of the encoded message will be a multiple of eight, and it will be possible to recover the message M unambiguously from the encoded message. (This padding rule is taken from RFC 1423 [RFC1423].)

eg.对5字节的数据使用PKCS#5填充
```
data: 01 02 03 04 05
```
需要填充3字节,填充数据位03
```
padding: 03 03 03
```
填充后 data||padding
```
data_padded: 01 02 03 04 05 03 03 03
```
## 8.PKCS #7 Padding
PKCS #7中也定义了一种填充方法,属于字节填充,填充方法和PKCS #5 Padding过程一致,区别是,PKCS #5 Padding的分组固定位8字节,PKCS #7 Padding的分组扩展成1~255之间,填充方法是一样的. 

>Some content-encryption algorithms assume the input length is a multiple of k octets, where k is greater than one. For such algorithms, the input shall be padded at the trailing end with k-(lth mod k) octets all having value k-(lth mod k), where lth is the length of the input. In other words, the input is padded at the trailing end with one of the following strings:
01 -- if lth mod k = k-1
02 02 -- if lth mod k = k-2
.
.
.
k k ... k k -- if lth mod k = 0
The padding can be removed unambiguously since all input is padded, including input values that are already a multiple of the block size, and no padding string is a suffix of another. This padding method is well defined if and only if k is less than 256.

因为只能用一个字节来表示填充的长度,所以分组长度最大只能是0xFF(255),分组长度不能大于256.因为填充过程同PKCS #5,这里不再举例了.