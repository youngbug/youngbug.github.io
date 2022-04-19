---
layout: post
title: Node.js使用ffi-napi,ref-array-napi,ref-struct-napi调用动态库
time: 2022年4月18日
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: Coding
tags: [cryptography, nodejs, javascript]
excerpt_separator:  <!--more-->
---

## 0x01 概述

使用electron开进行桌面程序的开发，似乎成了WEB前端开发人员转桌面程序开发的首选。近期有一些使用在electron中使用加密锁的需求，学习了一下在Node.js中通过ffi-napi模块调用动态链接库，把几款加密锁产品的动态库使用javascript封装了一下，实现了electron中使用加密锁功能。

开发过程中遇到了一些问题，踩了一些坑，这里总结记录一下。这里使用接口函数参数类型比较复杂的ROCKEY-ARM的动态链接库来进行开发。

<!--more-->

**NOTE:** javascript封装的ROCKEY-ARM接口模块源码，我已经分享出来，如果只是需要electron或者Node.js工程中使用ROCKEY-ARM的网友，可以直接使用。
``` shell
# 克隆
$ git clone https://github.com/youngbug/js-rockeyarm.git
```

## 0x02 准备

首先需要在node.js项目中安装调用动态链接库时需要依赖的模块**ffi-napi,ref-napi,ref-array-napi,ref-struct-napi**。

``` shell
npm install ffi-napi
npm install ref-napi
npm install ref-array-napi
npm install struct-napi
```

下面大概介绍一下这几个模块的用途：

- ffi-napi: 在javascript中调用动态链接库（.dll/.so），在Node.js中使用这个模块可以不写任何C/C++代码来创建一个对本地库的绑定。

- ref-napi: 这个模块定义了很多C/C++的常见数据类型，可以在声明和调用动态库的时候直接使用。

- ref-array-napi: 这个模块在Node.js中提供了一个数组的实现，在声明和调用函数中，所有的指针都可以声明成一个uchar数组。

- ref-struct-napi: 这个模块在Node.js中提供了一个结构体类型的实现。ROCKEY-ARM的函数很多参数都是结构体指针，如果声明称uchar的数组，那么传出的数据都是uchar数组，解析的时候不方便，需要自己拼接，除了麻烦，还要考虑字节序的问题。如果使用结构体，并定义一个结构体数组来作为指针传入，函数返回的结构体参数，就可以直接用结构体进行解析，会比较方便。

## 0x03 声明函数接口

ffi-napi支持Windows，Linux系统，所以.dll和.so都可以支持，在不同的操作系统下去加载不同的动态库文件就可以了。加载动态库的方法如下:

``` javascript
import { Library as ffi_Library } from 'ffi-napi'

libRockey = new ffi_Library('d:/rockey/x64/Dongle_d.dll',rockeyInterface)
```
Library()第一个参数是.dll的路径，Linux系统是.so的路径。第二个参数rockeyInterface是动态库导出函数的声明，ROCKEY-ARM的导出函数比较多，我单独拿出来定义。具体下面会讲到。

### 1 声明几个简单函数

首先从ROCKEY-ARM中找几个参数简单的函数来声明一下。

```c
typedef void * DONGLE_HANDLE;

DWORD WINAPI Dongle_Open(DONGLE_HANDLE * phDongle, int nIndex);

DWORD WINAPI Dongle_ResetState(DONGLE_HANDLE hDongle);

DWORD WINAPI Dongle_Close(DONGLE_HANDLE hDongle);

DWORD WINAPI Dongle_GenRandom(DONGLE_HANDLE hDongle, int nLen, BYTE * pRandom);
```
首先看一下上面几个接口用到的数据类型有:DONGLE_HANDLE,DWORD,DONGLE_HANDLE*,int,BYTE*这几种。
再看下ffi-napi支持的ref-napi支持的数据类型有以下类型：
``` javascript
void,int64,ushort,int,uint64,float,uint,long,double,int8,ulong,Object,uint8,longlong,CString,int16,char,byte,int32,uchar,size_t,uint32,short
```

参数这里应该用长度一致的数据类型，可以有以下匹配。

 |C类型|长度|ref-napi类型|说明|
 |-|-|-|-|
 |DONGLE_HANDLE|4/8|uint|C的定义是void*，是一个指针长度是4/8字节，用uint|
 |DONGLE_HANDLE*|4/8|ptrHandle|定义一个指向DONGLE_HANDLE的指针，用uint应该也是可以，但我没测试|
 |int|4|int||
 |BYTE*|4/8|prtByte|定义一个指向uchar的指针，用uint应该也是可以，但我没测试|

声明的写法如下：

```javascript
 const rockeyInterface = {
    'Dongle_Open' :             ['int', [ptrHandle, 'int']],
    'Dongle_ResetState' :       ['int', [ryHandle]],
    'Dongle_Close':             ['int', [ryHandle]],
    'Dongle_GenRandom' :        ['int', [ryHandle, 'int', ptrByte]]
 }
```
一个json，key是动态库导出函数名，比如'Dongle_Open'，value是个列表，第一个元素是返回值，第二个元素是参数。其中参数还是个列表。这个ref-napi中有适合类型的，直接写称具体类型即可，比如返回值DWORD和传入的长度int，我这里都用'int'。其他的参数我额外定义了句柄ryHandle、句柄的指针ptrHandle、字节的指针ptrByte。其中ryHandle，ptrryHandle，ptrByte的定义如下：

``` javascript
const refArray = require('ref-array-napi')

var ryHandle = refArray(ref.types.uint)
var ptrHandle = refArray(ryHandle)
var ptrByte = refArray(ref.types.uchar)
```

### 2 void*类型参数

DONGLE_HANDLE本质是void \*类型, void\* 类型最开始的时候妄图定义一个void的数组，然后用void数组来表示void*，然后发现报断言错误，数组不支持void类型。所以就直接用无符号数来表示void指针，在64位系统是8字节，32位系统是4字节，使用uint类型就可以了。DONGLE_HANDLE*。

### 3 结构体数组类型参数

在ROCKEY-ARM的函数中也有很多带参数的接口，比如：

```c
typedef struct {
	unsigned int  bits;                   
	unsigned int  modulus;				  
	unsigned char exponent[256];     

} RSA_PUBLIC_KEY;

typedef struct {
	unsigned int  bits;                   
	unsigned int  modulus;                
	unsigned char publicExponent[256];    
	unsigned char exponent[256];          

} RSA_PRIVATE_KEY;

typedef struct
{
	unsigned short  m_Ver;               
	unsigned short  m_Type;              
	unsigned char   m_BirthDay[8];       
	unsigned long   m_Agent;             
	unsigned long   m_PID;               
	unsigned long   m_UserID;            
	unsigned char   m_HID[8];            
	unsigned long   m_IsMother;          
  unsigned long   m_DevType;        

} DONGLE_INFO;

DWORD WINAPI Dongle_Enum(DONGLE_INFO * pDongleInfo, int * pCount);

DWORD WINAPI Dongle_RsaGenPubPriKey(DONGLE_HANDLE hDongle, WORD wPriFileID, RSA_PUBLIC_KEY * pPubBakup, RSA_PRIVATE_KEY * pPriBakup);
```

拿以上两个函数接口举例，Dongle_Enum中的第一个参数是一个指向DONGLE_INFO结构体的指针，运行后返回设备信息的列表，使用ROCKEY-ARM的时候需要通过枚举函数获得设备信息列表，然后比较产品ID或者硬件ID决定打开哪一个设备。为了方便从枚举函数返回的设备信息中方便的解析出产品ID或者硬件ID等信息，需要把DONGLE_INFO* pDongleInfo这个参数声明成一个结构体数组。Dongle_RsaGenPubPriKey()函数中有RSA_PUBLIC_KEY*,RSA_PRIVATE_KEIY*两个结构体指针参数，因为在这里一般用户并不需要解析RSA密钥中的n,d,e等分量，可以直接做作为一个字节数组，直接声明成上面的ptrByte类型即可。所以在声明如下：

``` javascript
const ref = require('ref-napi')
const refArray = require('ref-array-napi')
const StructType = require ('ref-struct-napi')

var dongleInfo = StructType({
    m_VerL:     ref.types.uchar,
    m_VerR:     ref.types.uchar,
    m_Type:     ref.types.ushort,
    m_BirthdayL:ref.types.uint32,
    m_BirthdayR:ref.types.uint32,
    m_Agent:    ref.types.uint32,
    m_PID:      ref.types.uint32,
    m_UserID:   ref.types.uint32,
    m_HIDL:     ref.types.uint32,
    m_HIDR:     ref.types.uint32,
    m_IsMother: ref.types.uint32,
    m_DevType:  ref.types.uint32
})

var ptrInt = refArray(ref.types.int)
var ryHandle = refArray(ref.types.uint)
var ptrHandle = refArray(ryHandle) 
var ptrDongleInfo = refArray(dongleInfo)
var ptrByte = refArray(ref.types.uchar)

const rockeyInterface = {
  'Dongle_Enum' :             ['int', [ptrDongleInfo, ptrInt]],
  'Dongle_RsaGenPubPriKey' :  ['int', [ryHandle, 'ushort', ptrByte, ptrByte]]
}
```

## 0x04 调用声明的函数

## 0x05 踩坑总结

### 1 结构体对齐的问题

### 2 无符号数和有符号数