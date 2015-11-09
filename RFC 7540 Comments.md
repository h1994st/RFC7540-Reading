RFC 7540 Comments
=================

这个Repo目的比较蛋疼：挑刺

## 1. Introduction

前辈协议的不足之处：

- HTTP/1.0中一个TCP连接上只能有一个Request
- HTTP/1.1中加入了Request Pipeling特性，但是这也只是部分地解决了并发的问题，仍然会导致head-of-line blocking的问题***(TODO: 不知道是这到底是个啥问题)***
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

两个标识符，`h2`和`h2c`

- `h2`: HTTP/2使用TLS，用于和服务器协商应用层所用协议，TLS中有两种相应的扩展，`NPN`和`ALPN`，`SPDY`时代多用`NPN`，RFC 7540规定使用`ALPN`，慢慢在迁移
