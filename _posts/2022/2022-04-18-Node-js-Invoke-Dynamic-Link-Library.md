---
layout: post
title: Node.js使用ffi-napi,ref-napi,ref-array-napi,ref-struct-napi调用动态库
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


## 0x03 声明函数接口

## 0x04 调用声明的函数

## 0x05 总结