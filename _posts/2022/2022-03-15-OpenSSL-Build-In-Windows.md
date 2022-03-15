---
layout: post
title: OpenSSL 在Windows上的编译
time: 2022年3月15日 星期二
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: Cryptography
tags: [cryptography, program]
excerpt_separator:  <!--more-->
---

近期工作中遇到一个早期的项目，使用了很古老的OpenSSL的密码库，在VC++ 2019等新环境下无法使用这么古老的Lib文件，现在的需要把旧的工程从VC++ 2005迁移到VC++ 2019上来。现在需要使用VC++ 2019编译一下OpenSSL。发现现在OpenSSL的编译跟早期版本的编译有了一点变化，记录一下编译的过程备忘。

<!--more-->

## 1.克隆代码

首先从OpenSSL项目的仓库里克隆出代码。

```shell
$ git clone git://git.openssl.org/openssl.git
```

## 2.安装编译工具

OpenSSL在Windows平台下，支持使用Visual C++、C++ Builder、MinGW、Cygwin等进行编译。这次我使用Visual C++ 2019来进行编译。

### 2.1 Perl

直接去OpenSSL源码中**NOTES-WINDOWS.md**中给的[ActiveState Perl](https://www.activestate.com/ActivePerl)链接下载安装。注册登录ActiveState Perl的网站，直接在线生成一个安装包和安装脚本，在Windows的控制台直接输入安装脚本完成在线安装。

安装完成之后在Windows控制台执行以下perl，看下perl是否被添加到环境变量里了，如果提示找不到perl，那么需要手工的把perl目录添加到环境变量里。

### 2.2 Visual C compiler

OpenSSL的文档建议使用最新的版本，旧版本的编译器可能会不再支持，我自己是用Visual C++ 2019里的。

### 2.3 Netwide Assembler(NASM)

按**NOTE-WINDOWS.md**中的介绍，去<https://www.nasm.us>这里下载，我下载的是<https://www.nasm.us/pub/nasm/releasebuilds/2.15.05/win64/nasm-2.15.05-installer-x64.exe>。

安装完成NASM后，在默认的安装目录中有一个**nasmpath.bat**，在控制台下执行这个批处理，可以把NASM的运行目录添加到环境变量中去。当然也可以自己手工把NASM的运行目录添加到环境变量的%path%中去。

## 3.编译工程

### 3.1 准备工作

使用管理员权限启动Windows控制台，先运行Visual C++的环境配置批处理**vcvarsall.bat**。这样就可以访问到编译时需要的namke.exe，cl.exe等，具体的可以看<https://docs.microsoft.com/cpp/build/building-on-the-command-line>

然后检查一下perl和nasm是否可以正确执行，如果提示找不到命令，需要把他们运行环境配置到环境变量中。

### 3.2 配置工程

在OpenSSL源码的根目录下，执行下面的命令。

|命令|说明|
|-|-|
|perl Configure VC-WIN32|编译32-bit的OpenSSL|
|perl Configure VC-WIN64A|编译64-bit的OpenSSL|

### 3.2 编译

如果3.2执行正确，执行下面步骤进行编译。
```shell
nmake

nmake test

nmake install
```
