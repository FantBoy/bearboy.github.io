---
layout: post
filename: high_concurrency_characteristics_of_nginx
title: Nginx优化之高并发特性
datetime: 2018-07-11 00:27:56
description: 
comments: true
tags:
 - nginx
categories:
 - nginx
 
---

Nginx采用多进程的运行模型，为了更好的支撑高并发特性，Nginx做了很多的优化，本文主要在学习了Nginx的基本运行原理后，整理的，Ngxin得以支撑高并发的一些优化点。主要包括数据处理、连接管理和异步请求处理。
<!--more-->

## Nginx采用多进程模型运行
Nginx其实也可以多线程，但主要采用的还是多进程：Master-Worker模型

Master:
 - 接收外界的信号
 - 向各Worker发送信号
 - 监控Worker进程的运行状态
 - 当Worker进程退出时，若存在异常退出，Master会自动重启Worker进程

Worker:
 - 处理各种基于网络的事件

 Nginx支持动态加载配置：service nginx reload （不停止服务加载最新配置，立即生效）
 TODO: reload的机制

## Nginx为何能处理高并发
Nginx能支持高并发特性，最核心的一点就是： Nginx采用异步非阻塞的方式运行来处理用户请求。
对比Apache：每个请求占用一个工作线程，大量的线程占用大量内存，上下文切换带来大量的CPU消耗，而Apache提供的异步非阻塞版本于Apache的一些模块存在冲突，不常用，也不建议使用。这也是Nginx相对于Apache最大的优势所在。

### Nginx建议将Worker的进程数设置为CPU的核数：
因为更多的进程只会导致更多的进程来竞争资源，从而带来上下文切换。Nginx为了更好的利用CPU多核的特性，提供了CPU情缘性的绑定选项。将Worker进程绑定到一个指定的核上，这样就不会因为进程间的切换带来cache的失效。

### 数据处理方面的优化
Nginx在`ngx_http_parse_request_line`阶段解析请求行时，为了提高处理效率，会采用状态机解析请求行数据，而且在进行`method`比较时，不会直接进行字符串的比较吗，而是将四个字节转换成一个整数，然后一次性整数比较，来减少CPU的指令数。

### keep-alive头域建立长连接
 - 当客户端请求的头域`connection`为`close`时，表明客户端需要关闭长连接。
 - 当`connection`为`keep-alive`时，则客户端需要建立长连接。（在HTTP1.1协议中，`connection`头域默认为`keep-alive`）

如果客户端请求为keepalive，则Nginx在输出完响应体后，会设置当前连接的keepalive属性，并设置等待时间`keepalive_timeout`，表明等待客户端下一次请求到来时的最大等待时间，若该值为0，则表明关闭keepalive属性（强制关闭）。

如果服务端最后决定时打开keepalive，那么服务端会在响应头里包含`connection`头域，值同样为`keep-alive`，否则为`close`。若服务端返回了`close`，则Nginx在响应完数据后，会主动关闭连接。

 > 对于高并发的业务场景，服务端关闭keepalive后，主动关闭连接会产生大量的`time-out`状态的socket，影响Nginx服务的整体性能。一般来说，客户端的一次访问，会造成后台Nginx的多次访问，这是，打开`keepalive`带来的优势。


### pipe
`pipeline`, HTTP1.1协议版本引入的新特性（流水线作业），是`keep-alive`的一次升华，是基于长连接实现的。目的就是基于长连接完成多次请求。

如果客户端发起多次请求：
 - keepalive： 第二个请求必须要在第一个请求的响应接收完成后，才能发起（得到两次请求的响应的总时间至少为2倍RTT）
 - pipeline：客户端不必等第一个请求请求处理完成，就可以直接发送第二个请求。得到两个请求响应的时间可以达到1倍RTT








