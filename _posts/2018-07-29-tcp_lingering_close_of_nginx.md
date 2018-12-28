---
layout: post
filename: tcp_lingering_close_of_nginx
title: Nginx优化之关于连接的关闭 
datetime: 2018-07-29 17:11:29
description: 
comments: true
tags:
 - nginx
categories:
 - nginx
 
---

Nginx在设计连接关闭的相关特性的时候，引入了`lingering_close`特性，其主要作用是让服务器端在连接存在异常断开的时候，能有充足的时间来处理客户端数据，更好的保护客户端特性。以消耗更多的额外资源（超时前，连接一直存在）换来数据的完整发送和通信可靠。
<!-- more -->

## Nginx连接关闭延时 lingering_close
当Nginx关闭连接时，并非立刻关闭连接，而是先关闭tcp连接的写，等一段时间后再关闭tcp连接的读。
### 场景描述
Nginx在接收客户端请求时，若因为客户端或者服务器端的异常问题，需要立即向客户端响应错误码信息。而当Nginx在返回错误码信息之后，大部分的场景都需要在返回后关闭tcp连接。
 - Nginx发送异常信息给客户端：`Write()`写数据到`write_buff`，待发送给客户端
 - Nginx调用`close()`关闭tcp连接
 

### 连接关闭流程
当Nginx直接调用`close()`关闭连接时，内核会首先检查tcp的`read_buff`里还有没有客户端发送过来的留在内核态没有被用户态进程读取的数据。如果有，则向客户端发送`RST`报文来关闭tcp连接，并丢弃`write_buff`里的数据。如果没有，则等待`write_buff`里的数据发送完毕后，再经过正常的4次分手报文断开tcp连接。

>在调用`close()`系统调用时，若之前`write()`到`write_buff`里的错误信息数据还没有完全发送给客户端，且`read_buff`里还有数据没有读取。这时`close()`调用会导致服务器向客户端发送`RST`报文，客户端则在未收到错误信息的情况下，接收到了服务端发送来的`RST`报文，而关闭了连接。

### lingering_close机制的引入
Nginx通过`lingering_close`设置来判断是否打开该特性。当Nginx关闭连接时，只关闭tcp连接的写，在一段时间`lingering_timeout`之后，再关闭tcp连接的读。而`linger_close`的存在意义，就是读取客户端发送过来的，还未处理的剩下的数据。

> 正常的客户端，在接收到了错误信息之后，会关闭掉连接，服务端的连接则会在超时前关闭

`lingering_close`引入了两个超时时间来保证上述操作：
 - lingering_timeout：读超时时间。若服务器端在`lingering_timeout`内未收到客户端的数据，则关闭tcp连接的读，关闭连接。
 - lingering_time：总的读取时间。在服务端关闭tcp连接的写之后，保留socket的有效时间，在时间`lingering_time`的时间内，客户端需要将数据全部发送给服务端，否则，服务端会在`lingering_time`后关闭tcp连接。
 

 









