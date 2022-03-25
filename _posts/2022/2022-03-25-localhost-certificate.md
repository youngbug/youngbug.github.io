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

事情的缘由是客户的Web应用程序需要和自己公司开发的原生应用程序(一个websockets服务)通信，原生应用程序需要分发到客户的最终用户。原生websockets服务程序是安装部署在每一个最终客户的计算机上的，Web应用在请求ws服务地址是**ws://127.0.0.1**或者**ws://localhost**，由于之前给客户提供的Web应用通信示例都是HTTP页面，现在客户要使用HTTPS。在HTTPS页面上HTTP的请求都会被拒绝，浏览器禁止向非安全链接发送XMLHTTPRequest(XHR)或者WebSockets请求。这个称为混合内容阻断。要与Web应用程序通信，原生的WebSockets服务应用程序需要使用wss协议。

<!--more-->

## 能否继续使用ws协议苟且

如果可以让客户使用HTTPS的Web应用，继续使用ws协议和我们的WebSockets服务继续通信，这个是最简单了。比如只是在WebSockets服务或者Web应用里改几行代码，修改一些配置，就可以不改为wss协议，继续使用ws协议苟且是最好的了。于是做了一番调研。

现在的浏览器认为**http://127.0.0.1:1234/**是一个["潜在的可信赖来源(potentially trustworthy origin)"](https://www.w3.org/TR/secure-contexts/#is-origin-trustworthy)。因为他指向一个回环地址，发送到**127.0.0.1**的流量肯定不会离开你的计算机，因此它不受网络拦截的影响。这意味着客户的Web应用使用了HTTPS，我们在**127.0.0.1**上提供的原生应用程序是个Web服务，两者可以使用XHR进行通信。但是localhost的存在没有实际映射到回环地址(loopback address)的情况，所以发送**localhost**的流量并不享受**127.0.0.1**的待遇，详情参考[Let 'localhost' be localhost.](https://datatracker.ietf.org/doc/html/draft-ietf-dnsop-let-localhost-be-localhost-02)。

**localhost**不能享受**127.0.0.1**的待遇似乎也不是什么问题，我使用**127.0.0.1**呗，但是非常不幸的是WebSockets的这两个名字都不能享受特殊待遇，浏览器都会把到他们的流量拦截下来。所以我们自己的原生WebSockets服务，向不升级到wss协议，继续用ws苟且，是不可能的了。

## 能否向CA申请一个localhost证书

使用wss协议就需要数字证书，因为服务器是localhost，所以需要一个颁发给localhost的数字证书。这个就存在一个问题了，因为没有谁会唯一的拥有"localhost"，所以既然没有谁能证明自己拥有"localhost"，也就没有CA会给"localhost"颁发证书了。

因为CA无法直接给localhost颁发证书，那么就只有绕过localhost或者绕过CA两种方案了。

### 绕过localhost的限制

在全局DNS中设置域名(比如**localhost.abcd.com**)解析为**127.0.0.1**，向CA申请**localhost.abcd.com**的证书。证书和对应的私钥和WebSockets服务应用一起分发，并将Web应用中请求**ws://127.0.0.1**修改为**wss://localhost.abcd.com**。

当然这种方案私钥是随WebSockets服务分发给每一个最终用户的，私钥存在泄露的风险，所以安全性没有保证。另外由于私钥可能泄露，当CA发现了你的证书的私钥泄露，CA就会按照规定吊销这个证书，现实中也大量存在原生应用程序把私钥分发出去而被CA吊销证书的例子。所以这个方案安全上有风险，可用性上也存在问题，当CA吊销了证书之后，就又不能用了。

所以这种方法，我个人是觉得不太好。

### 不用CA签发的证书

没有CA签发的证书，那就使用自己制作的自签名证书呗。我们自己做的自签名证书和CA颁发的证书区别不就是没人信任么，但是因为需要在最终用户的计算机上安装WebSockets服务，所以再安装一个根证书也是可以的(当然需要用户同意了)。

自己为localhost生成一个私钥和自签名证书，使用openssl命令即可。Linux下最简单，直接输入命令即可，Windows下需要下载安装包，可以到[OpenSSL Wiki](https://wiki.openssl.org/index.php/Binaries)上推荐的链接下载。

opensll生成自签名证书的最简单命令如下：

```shell
openssl req -x509 -out localhost.crt -keyout localhost.key \
  -newkey rsa:2048 -nodes -sha256 \
  -subj '/CN=localhost' -extensions EXT -config <( \
   printf "[dn]\nCN=localhost\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:localhost\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
```

然后就可以使用**localhost.crt**和**localhost.key**来配置WebSockets了，然后在浏览器本地受信任的根证书列表中安装**localhost.crt**即可。

当然也可以先生成本地根证书，然后颁发根证书签名的实体证书，然后让最终用户导入自签名的根证书，不用再导入实体证书了。