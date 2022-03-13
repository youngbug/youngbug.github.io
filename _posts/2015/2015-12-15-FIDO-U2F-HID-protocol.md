---
layout: post
title: FIDO U2F设备的HID协议
time: 2015年12月15日 星期二
author: Zhao Yang(cnrgc@163.com)
location: 北京
pulished: true
category: SmartCard
tags: [cryptography, smartcard, fido, u2f, hid]
excerpt_separator:  <!--more-->
---
# FIDO概述

FIDO（Fast IDentity Online）联盟成立于2012年，FIDO联盟通过定义出一套开放、可扩展、可协同的技术规范，来改变现有在线认证方式，减少认证用户时对密码（password）的依赖。FIDO有两套规范:U2F和UAF。

<!--more-->

## 无密码的UAF（Universal Authentication Framework）

- 用户携带含有UAF的客户设备
- 用户出示一个本地的生物识别特征或者PIN
- 网站额可以选择是否保存密码

用户选择一个本地的认证方案（例如按一下指纹、看一下摄像头、对麦克说话，输入一个PIN等)把他的设备注册到在线服务上去。只需要一次注册，之后用户再需要去认证时，就可以简单的重复一个认证动作即可。用户在进行身份认证时，不在需要输入他们的密码了。UAF也允许组合多种认证方案，比如指纹+PIN。

## 第二因子的U2F(Universal 2nd Factor) 

- 用户携带U2F设备，浏览器支持这个设备
- 用户出示U2F设备
- 网站可以使用简单的密码（比如4个数字的PIN）

FIDO U2F认证，国内的文章一般翻译成FIDO两步认证。U2F是在现有的用户名+密码认证的基础之上，增加一个更安全的认证因子用于登录认证。用户可以像以前一样通过用户名和密码登录服务，服务会提示用户出示一个第二因子设备来进行认证。U2F可以使用简单的密码（比如4个数字的PIN）而不牺牲安全性。

U2F出示第二因子的形式一般是按一下USB设备上的按键或者放入NFC。

# U2F HID协议

UAF先放一边，U2F的工作流程比较简单，具体的可以看FIDO联盟官网。下面主要说下U2F HID协议。

首先要明确一下U2FHID协议不是U2F的应用层协议，是描述U2F的消息如何通过HID传输的底层协议，U2F应用层的协议在U2F Raw Message中定义。U2FHID协议可以支持在大多数平台上直接使用而不需要安装驱动，可以支持多应用并发访问设备。

## 并发和通道

U2FHID设备处理多客户端，比如多个应用通过HID栈访问单个资源，每个客户端都可以和U2FHID设备通过一个逻辑通道（logical channel）进行通讯，每个客户端都使用一个唯一的32bit通道ID来判断用途。通道ID由U2F设备来进行分配，确保唯一。产生通道ID的算法由U2FHID设备的厂商规范定义，FIDO的U2FHID协议中不进行定义。

通道ID 0是保留的，0xFFFFFFFF也是保留给广播命令的。

## 消息和包结构

包(Packets)分为两类，初始化包（initialization packets）和附加包（continuation packets）。就像initialization packets这个名字一样，每个应用层消息的第一包都是initialization packet,也是一个事物的开始。如果整个应用层消息不能通过一个包下发，就需要一个或者多个continuation packet来发送了，直到把所有消息发完。

一个应用层消息从主机发送到设备叫做请求(request)，从设备返回给主机的叫做响应(response)。请求和响应消息是相同的结构，一个事勿从一个请求的initialization packet开始，截止于一个响应的最后一个包。包的长度永远是固定的大小，一个包中的有的字节并没有被使用到，没有使用的字节需要设置为0。


**初始化包的定义**

|偏移	|长度	|名称	|描述|
|-|-|-|-|
|0|	4|	CID|	通道ID|
|4	|1	|CMD	|命令ID|
|5	|1	|BCNTH	|发送数据长度的高位|
|6	|1	|BCNTL	|发送数据长度的低位|
|7	|(s-7)	|DATA	|发送数据（s等于包的固定长度）|

---

**附加包的定义**

|偏移	|长度	|名称	|描述|
|-|-|-|-|
|0	|4	|CID	|通道ID|
|4	|1	|SEQ	|包的顺序，0x00..0x7F|
|5	|(s-5)	|DATA	|发送数据（s等于包的固定长度）|

---

如果一个应用层消息长度小于等于s-7，那么一个包就可以发完，如果一个比较大的应用层消息，需要拆分成一个或者多个附加包，第一个附加包的SEQ为0，每次附加包的SEQ值增加1，最大到0xFF。 USB全速设备的包长度的最大值是64字节，这样最大的一个应用层消息可以发送的长度就是，64-7+128*（64-5）=7609字节。

下面是一个初始化包和附加包的C的定义
```c
typedef struct {
  uint32_t cid;                        // Channel identifier
  union {
    uint8_t type;                      // Frame type - b7 defines type
    struct {
      uint8_t cmd;                     // Command - b7 set
      uint8_t bcnth;                   // Message byte count - high part
      uint8_t bcntl;                   // Message byte count - low part
      uint8_t data[HID_RPT_SIZE - 7];  // Data payload
    } init;
    struct {
      uint8_t seq;                     // Sequence number - b7 cleared
      uint8_t data[HID_RPT_SIZE - 5];  // Data payload
    } cont;
  };
} U2FHID_FRAME;
```

## 同步

搞USBKEY就不能不说到同步，U2FHID也一样，一个事务永远分为三个阶段：消息从主机发送到设备，设备处理消息，响应从设备返回到主机，U2FHID的事务必须是原子的，一个事务一旦开始，不能被其他应用中断。

##U2FHID 命令

还是要明确一下，下面介绍的是U2FHID协议中的命令，不是U2F应用层的命令，U2F应用层的命令在U2F Raw Message中定义，并使用U2F_MSG命令发送。

### 1. U2FHID_MSG

这个指令是用来发送U2F应用层消息的到FIDO设备的指令，数据域data中的值就是U2F的应用层指令。

### 2.U2FHID_INIT

这个指令是应用请求FIDO设备分配一个唯一的32bit的CID，这个CID将会在应用剩下的时间中使用到。请求FIDO设备分配一个新的逻辑通道，请求的应用需要使用广播通道U2FHID_BROADCAST_CID,设备会使用广播通道在响应中重新返回一个通道ID。

### 3.U2FHID_PING

用来调试用的，发送一个事务到设备。

### 4.U2FHID_ERROR

只用于响应消息。

### 5.U2FHID_WINK

这个指令是可选的，如果设备有LED灯，可以让灯闪一下或者其他类似的行为，主要是用于提示用户。

可以看出来，U2FHID协议最基本的就是使用上面的U2FHID_FRAME这个结构来和FIDO设备进行通讯，下面这个函数就是把不同的U2FHID指令和数据域封装成U2FHID_FRAME的格式下发到USB设备中。

```c
u2fh_rc
u2fh_sendrecv (u2fh_devs * devs, unsigned index, uint8_t cmd,
	       const unsigned char *send, uint16_t sendlen,
	       unsigned char *recv, size_t * recvlen)
{
  int datasent = 0;
  int sequence = 0;
  struct u2fdevice *dev;
 
  if (index >= devs->num_devices || !devs->devs[index].is_alive)
    {
      return U2FH_NO_U2F_DEVICE;
    }
 
  dev = &devs->devs[index];
 
  while (sendlen > datasent)
    {
      U2FHID_FRAME frame = { 0 };
      {
	int len = sendlen - datasent;
	int maxlen;
	unsigned char *data;
	frame.cid = dev->cid;
	if (datasent == 0)
	  {
	    frame.init.cmd = cmd;
	    frame.init.bcnth = (sendlen >> 8) & 0xff;
	    frame.init.bcntl = sendlen & 0xff;
	    data = frame.init.data;
	    maxlen = sizeof (frame.init.data);
	  }
	else
	  {
	    frame.cont.seq = sequence++;
	    data = frame.cont.data;
	    maxlen = sizeof (frame.cont.data);
	  }
	if (len > maxlen)
	  {
	    len = maxlen;
	  }
	memcpy (data, send + datasent, len);
	datasent += len;
      }
 
      {
	unsigned char data[sizeof (U2FHID_FRAME) + 1];
	int len;
	data[0] = 0;
	memcpy (data + 1, &frame, sizeof (U2FHID_FRAME));
	if (debug)
	  {
	    fprintf (stderr, "USB send: ");
	    dumpHex (data, 0, sizeof (U2FHID_FRAME));
	  }
 
	len = hid_write (dev->devh, data, sizeof (U2FHID_FRAME) + 1);
	if (debug)
	  fprintf (stderr, "USB write returned %d\n", len);
	if (len < 0)
	  return U2FH_TRANSPORT_ERROR;
	if (sizeof (U2FHID_FRAME) + 1 != len)
	  return U2FH_TRANSPORT_ERROR;
      }
    }
 
  {
    U2FHID_FRAME frame;
    unsigned char data[HID_RPT_SIZE];
    int len = HID_RPT_SIZE;
    int maxlen = *recvlen;
    int recvddata = 0;
    short datalen;
    int timeout = HID_TIMEOUT;
    int rc = 0;
 
    while (rc == 0)
      {
	if (debug)
	  {
	    fprintf (stderr, "now trying with timeout %d\n", timeout);
	  }
	rc = hid_read_timeout (dev->devh, data, len, timeout);
	timeout *= 2;
	if (timeout > HID_MAX_TIMEOUT)
	  {
	    rc = -2;
	    break;
	  }
      }
    sequence = 0;
 
    if (debug)
      {
	fprintf (stderr, "USB read rc read %d\n", len);
	if (rc > 0)
	  {
	    fprintf (stderr, "USB recv: ");
	    dumpHex (data, 0, rc);
	  }
      }
    if (rc < 0)
      {
	return U2FH_TRANSPORT_ERROR;
      }
 
    memcpy (&frame, data, HID_RPT_SIZE);
    if (frame.cid != dev->cid || frame.init.cmd != cmd)
      {
	return U2FH_TRANSPORT_ERROR;
      }
    datalen = frame.init.bcnth << 8 | frame.init.bcntl;
    if (datalen + datalen % HID_RPT_SIZE > maxlen)
      {
	return U2FH_TRANSPORT_ERROR;
      }
    memcpy (recv, frame.init.data, sizeof (frame.init.data));
    recvddata = sizeof (frame.init.data);
 
    while (datalen > recvddata)
      {
	timeout = HID_TIMEOUT;
	rc = 0;
	while (rc == 0)
	  {
	    if (debug)
	      {
		fprintf (stderr, "now trying with timeout %d\n", timeout);
	      }
	    rc = hid_read_timeout (dev->devh, data, len, timeout);
	    timeout *= 2;
	    if (timeout > HID_MAX_TIMEOUT)
	      {
		rc = -2;
		break;
	      }
	  }
	if (debug)
	  {
	    fprintf (stderr, "USB read rc read %d\n", len);
	    if (rc > 0)
	      {
		fprintf (stderr, "USB recv: ");
		dumpHex (data, 0, rc);
	      }
	  }
	if (rc < 0)
	  {
	    return U2FH_TRANSPORT_ERROR;
	  }
 
	memcpy (&frame, data, HID_RPT_SIZE);
	if (frame.cid != dev->cid || frame.cont.seq != sequence++)
	  {
	    fprintf (stderr, "bar: %d %d %d %d\n", frame.cid, dev->cid,
		     frame.cont.seq, sequence);
	    return U2FH_TRANSPORT_ERROR;
	  }
	memcpy (recv + recvddata, frame.cont.data, sizeof (frame.cont.data));
	recvddata += sizeof (frame.cont.data);
      }
    *recvlen = datalen;
  }
  return U2FH_OK;
}
```
这个函数搞定，基本上U2FHID协议中定义的命令就都可以搞定了。下一篇暂定写U2F Raw Message中定义的U2F应用层的协议。