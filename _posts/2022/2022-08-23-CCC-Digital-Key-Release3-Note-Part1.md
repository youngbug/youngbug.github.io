---
layout: post
title: CCC 数字钥匙Release 3学习笔记 Part1
time: 2022年8月23日
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: cryptography
tags: [cryptography]
excerpt_separator:  <!--more-->
---

## 0x00 前言

2022年8年开始离开了从事14年的密码行业，去了汽车行业，当然在全新的行业里，还是去做密码相关的东西。最近开始学习目前CCC正在推广的的数字钥匙规范。

## 0x01 介绍

CCC数字钥匙技术规范(Digital Key Technical Specification)Release 3详细描述了一个数字钥匙生态系统，这个数字钥匙生态系统使用标准化数字钥匙applet、标准化车辆访问协议(vehicle access protocol)和一个可扩展的架构来支持不同车辆OEM和设备OEM进行数字钥匙服务的大规模开发。
Release 3是向前兼容Release 2的，但是Release 3不支持Release 1。Release 1可以独立与Release2和3部署。
数字钥匙技术规范Release 3基于BLE/UWB 或者NFC作为低层的无线通信技术来启用数字钥匙服务。数字钥匙服务管理框架被设计为和无线通信技术无关的，使得框架可以支持其他技术。

## 0x02 系统架构

