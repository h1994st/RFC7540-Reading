RFC 7540 Comments
=================

这个Repo目的比较蛋疼：挑刺

本文中会使用几种标记，方便之后查找：

- ***???***，表示有疑惑
- ***DONE-yyyy.MM.dd***，这个针对***???***，用于解释
- ***!!!***，表示可以挖掘的点
- ***ERR: xxx(type)***，表示RFC 7540中出现的错误类型
- MUST、MUST NOT等等，表示强调，含义与RFC一致

## 1. Introduction

前辈协议的不足之处：

- HTTP/1.0中一个TCP连接上只能有一个Request
- HTTP/1.1中加入了Request Pipeling特性，但是这也只是部分地解决了并发的问题，仍然会导致head-of-line blocking的问题 ***(???: 不知道是这到底是个啥问题)***
- HTTP头部信息有冗余，导致不必要的网络开销，这样会使得TCP拥塞窗口很快就被填满，从而导致延迟

HTTP/2就是来解决这些问题，改进：

- 没有改变HTTP的语义
- 允许在同一个TCP连接上Request和Response交错
- 此外还对头部编码方式做了改变，二进制编码，以及压缩算法
- Request可以设置优先级，保证重要的请求能优先处理

这样使用的TCP连接数减少了许多

## 2. HTTP/2 Protocol Overview

没什么好挑刺的，提到了几个重要概念：

- HTTP/2中基本单位是Frame，有各种各样的类型
- Multiplexing
- Flow control and prioritization
- Server Push, PUSH_PROMISE frame
- Header Compression

### 2.1 Document Organization

(略)

### 2.2 Conventions and Terminology

"MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL"

这些词表示这份标准文档对不同内容具体实现的要求程度，值得注意一下

## 3. Starting HTTP/2

URI、端口都和HTTP/1.1统一

### 3.1 HTTP/2 Version Identification

两个标识符，`h2`和`h2c`，其实不止这么一些，因为正式版本的HTTP/2协议出来之前还有许多草案，所以还存在许多形如`h2-*`的标识符，表示不同的草稿版本

- `h2`: HTTP/2 over TLS，用于和服务器协商应用层所用协议，TLS中有两种相应的扩展，`NPN`和`ALPN`，`SPDY`时代多用`NPN`，RFC 7540规定使用`ALPN`，慢慢在迁移
- `h2c`: 不加密传输，用于HTTP/1.1向HTTP/2升级时在头部中使用，或者用于HTTP/2 over TCP

***(???: 对于第二点还是挺疑惑的，因为大部分的服务器都没有实现HTTP/2 over TCP，不知道h2c该怎么使用)***

### 3.2 Starting HTTP/2 for "http" URIs

HTTP/1.1 => HTTP/2 的升级方式，标识符`h2c`

```
GET / HTTP/1.1
Host: server.example.com
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c
HTTP2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>
```
***!!!: 文档中说如果Request携带payload，必须在客户端能发送HTTP/2 Frame之前全部发完，这意味着一个大的请求可能阻塞连接，直到它全部发完***

***???: concurrency of an initial request with subsequent requests is important是啥意思= =***

对于上文那个请求，服务器可能有两种响应：

1. 服务器支持HTTP/2

    ```
    HTTP/1.1 200 OK
    Content-Length: 243
    Content-Type: text/html

    ...
    ```

    此时，服务器必须(MUST)忽略`h2`这个标识符

2. 服务器不支持HTTP/2

    ```
    HTTP/1.1 101 Switching Protocols
    Connection: Upgrade
    Upgrade: h2c

    [ HTTP/2 connection ...
    ```

    通过101来响应，这之后就可以发送HTTP/2 Frame了

    ***!!!: 这些Frame中必须包含对最开始那个升级请求的响应***

    由服务器发送的第一个Frame必须(MUST)是一个包含SETTINGS Frame的connection preface

    客户端收到101的响应后，必须(MUST)发送一个包含SETTINGS Frame的connection preface ***(???: connection preface具体指什么)***

    之前发出去的那个用于升级的请求，被赋予一个Stream编号，为1，默认优先级(weight=16)。此时，Stream 1处于[half-closed(local)](https://tools.ietf.org/html/rfc7540#section-5.1)状态，因为已经发送了请求。当开始HTTP/2连接后，Stream 1用于response

#### 3.2.1 HTTP2-Settings Header Field

在用于升级的请求中，必须(MUST)包含`HTTP2-Settings`字段

***(???: [token68](https://tools.ietf.org/html/rfc7235#section-2.1)是个啥)***

如果没有`HTTP2-Settings`字段或者有多个该字段，服务器绝对不能(MUST NOT)升级到HTTP/2

`HTTP2-Settings`字段的内容是SETTINGS Frame的payload，以`base64`编码

***(???: a client sending the HTTP2-Settings header field MUST also send "HTTP2-Settings" as a connection option in the Connection header field to prevent it from being forwarded，这句话没懂)***

***(DONE-2015.11.09: `Connection`是HTTP头部中的一个字段，详见[RFC 7230, Section6.1](https://tools.ietf.org/html/rfc7230#section-6.1))***

***(???: 但是关于转发那个还是没怎么明白)***

这个升级请求中的`HTTP2-Settings`相关信息会和任何SETTINGS Frame一样对待，并且无需显示ACK，因为之后的那个101响应就是一个隐式的ACK

### 3.3 Starting HTTP/2 for "https" URIs

使用ALPN进行应用层协议协商，标识符为`h2`，协商是属于TLS的流程

`h2c`协议标识符绝对不能(MUST NOT)在使用TLS的时候由客户端发送，或是被服务器所选择

一旦协商完毕，客户端和服务器必须(MUST)发送一个connection preface

### 3.4 Starting HTTP/2 with Prior Knowledge

***(???: [ALT-SVC](https://tools.ietf.org/html/rfc7540#ref-ALT-SVC)是啥......)***

客户端必须(MUST)发送connection preface，然后可能(MAY)立即向服务器发送HTTP/2 Frame

服务器可以通过connection preface的存在来识别出这些连接

这仅仅只影响HTTP/2 connections over cleartext TCP的建立

HTTP/2 over TLS则必须(MUST)使用TLS中的协议协商

同样地，服务器也必须(MUST)发送connection preface

如果没有更多其它的信息，这个prior support for HTTP/2并不是一个很强的信号表示这个服务器会在之后的连接中支持HTTP/2

### 3.5 HTTP/2 Connection Preface

客户端和服务器各自发送不同的connection preface

客户端的connection preface以24字节开头：`0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a`，翻译成字符串就是：`PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`。这一串之后必须(MUST)跟上SETTINGS Frame，但是这个SETTINGS Frame可能(MAY)是空的

客户端在以下两种情况时，发送connecion preface：

- 收到101响应
- 作为TLS连接的第一批application data octets

如果此前知道服务器支持HTTP/2，那么客户端在HTTP/2连接建立的时候发送connection preface

***(???: connection preface的目的是啥)***

服务器的connection preface则由一个有可能为空的SETTINGS Frame构成，这个SETTINGS Frame必须(MUST)是服务器在HTTP/2连接上发出的第一个Frame

当收到来自某一端的作为connection preface一部分的SETTINGS Frame后，必须(MUST)在发送完connection preface后ACK

为了避免不必要的延迟，客户端允许在发送connection preface之后立即发送其他Frame，不需要等待接受服务器的connection preface

值得注意的是，服务器的connection preface里的SETTINGS Frame可能包含一些很重要的参数，可能会改变此后两者的交流的一些细节

在客户端收到SETTINGS Frame后，客户端被期望去遵守其中的参数

***(!!!: 如果不遵守呢？比如最大超过最大并发量，直接超过窗口大小等等)***

在某些配置情况下，服务器传送SETTINGS Frame的时间早于客户端发送额外Frame的时间

***(???: 原文中的providing an opportunity to avoid this issue是指哪个issue)***

客户端和服务器必须(MUST)将一个不合法的connection preface视为***ERR: connection error(PROTOCOL_ERROR)***，此时一个GOAWAY Frame可能被触发，因为一个非法的connection preface意味着某一终端不是在使用HTTP/2

## 4. HTTP Frames

一旦HTTP/2连接被建立，终端之间就可以开始互相发送Frame了

### 4.1 Frame Format

格式如下：

```
+-----------------------------------------------+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+
```

Frame header总长9字节，下面是每个字段的详解：

|Field|Length|Note|
|:---:|:----:|:---|
|Length|24 bits|仅仅是payload的长度，单位字节，不包括Frame header自身的9字节。被视为一个unsigned 24-bit integer。大于2^14 (16384)的值绝对不能(MUST NOT)被设置，除非接受端有一个很大的`SETTINGS_MAX_FRAME_SIZE`值|
|Type|8 bits|RFC 7540中定义了一些类型，但是还是有扩展空间的，任何实现都必须(MUST)忽视或者取消认可未知的Frame type|
|R|1 bit|保留位，该位暂无意义，这一位在发送的时候必须(MUST)保持为`0x0`，接受的时候必须(MUST)忽视|
|Stream Identifer|31 bits|被视为一个unsigned 31-bit integer，`0x0`是一个特殊的值，为一些与整个连接有关的Frame所预留的，而不是为某一个特定的Stream|

***(!!!: 那么似乎也是有可能可以将Length设置得很大，RFC 7540中并没有说大了会触发什么错误)***

***(!!!: 此外根据Dr. Luo的建议，此处Length和Payload的真实长度也可以不相符，看看会发生什么)***

### 4.2 Frame Size

Frame payload的大小被发送方所声明的`SETTINGS_MAX_FRAME_SIZE`值所限制，这是在SETTING Frame中可以设置的一个字段，合法的值范围是[2^14 (16384), 2^24-1 (1677215)]

***(!!!: 似乎没法超越上界，下界还是可以试下的)***

所有的实现都必须(MUST)有能力接收并处理大小为2^14字节(还需在加上9字节的header)以下的Frame，当描述Frame size时并不包含Frame header的大小

有些特殊类型的Frame对于payload大小有额外的要求

以下情况，终端必须(MUST)发出***ERR: (FRAME_SIZE_ERROR)***：

- Frame的长度超过了`SETTINGS_MAX_FRAME_SIZE`所声明的大小
- Frame的长度超过了某些特殊类型的Frame的限制
- Frame的长度太小，不足以包含必须的数据

如果一个Frame大小错误可以改变整个连接的状态，那么这个错误必须(MUST)被视为***ERR: connection error***。这包括任何带有header block的Frame(HEADERS Frame, PUSH_PROMISE Frame, CONTINUATION Frame)、SETTINGS Frame、所有stream identifier为0的Frame

终端无需用完一个Frame中一切可用的空间。如果使用一些比大小上限小的大小，响应状况会有所提高。发送大的Frame会导致延迟，尤其是对于那些时间敏感的Frame(RST\_STREAM Frame, WINDOW\_UPDATE Frame, PRIORITY Frame)，这会影响性能

### 4.3 Header Compression and Deconpression

和HTTP/1一样，HTTP/2中的每一个header field都是一个键值对。header field在请求和响应中都会有，在server push中也会有

header list里包含0个或多个header field。当在被传输的时候，一个header list被序列化放入一个header block
























