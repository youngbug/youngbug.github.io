---
layout: post
title: localhost证书
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: Cryptography
tags: [cryptography, certificate]
excerpt_separator:  <!--more-->
---

事情的缘由是客户的Web应用程序需要和自己公司开发的原生应用程序(一个websockets服务)通信，原生应用程序需要分发到客户的最终用户。原生websockets服务程序是安装部署在每一个最终客户的计算机上的，Web应用在请求ws服务地址是ws://127.0.0.1//或者ws://localhost/，由于之前给客户提供的Web应用通信示例都是HTTP页面，现在客户要使用HTTPS。在HTTPS页面上HTTP的请求都会被拒绝，浏览器禁止向非安全链接发送XMLHTTPRequest(XHR)或者WebSockets请求。这个成为混合内容阻断。要与Web应用程序通信，原生的WebSockets服务应用程序需要使用wss协议。

<!--more-->

使用wss协议需要证书，因为服务是localhost，所以需要一个办法给localhost的数字证书。这个就存在一个问题了，因为没有谁会唯一的拥有"localhost"，所以既然没有谁能证明自己拥有"localhost"，也就没有CA会给"localhost"办法证书了。