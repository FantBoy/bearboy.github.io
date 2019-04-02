---
layout: post
title: Nginx特性之Keepalive连接保持
datetime: 2019-04-02 22:21:32
description: Nginx特性之Keepalive连接保持
comments: true
tags:
 - Nginx
categories:
 - Nginx
---

Nginx关于keepalive连接保持的特性，实际上就是在一次TCP连接中，可以持续处理多个客户请求，而不断开连接。通过该机制可以减少TCP连接的建立次数，减少`TIME_WAIT`的状态连接。从而增加服务的吞吐量和整体服务质量。但是，长时间的TCP连接会导致系统资源被长时间占用，浪费资源，所以在实际使用的时候，还需要为keepalive设置合理的`timeout`。

## Nginx关于Keepalice的配置项说明

>Syntax:	keepalive_timeout timeout [header_timeout];
>Default:	
>keepalive_timeout 75s;
>Context:	http, server, location
>The first parameter sets a timeout during which a keep-alive client connection will stay open on the server side. The zero value disables keep-alive client connections. The optional second parameter sets a value in the “Keep-Alive: timeout=time” response header field. Two parameters may differ.
>
>The “Keep-Alive: timeout=time” header field is recognized by Mozilla and Konqueror. MSIE closes keep-alive connections by itself in about 60 seconds.

## timeout的含义
一个HTTP请求会生成一条TCP连接，其在完成了最后一次响应后，该连接还会存在一段时间而不被销毁。这个时间就是`keepalive_timeout`连接最长保持时间。在这期间：
1. 在`keepalive_timeout`时间内，若有新的请求进来，使用了该连接，timeout时间将会被刷新，重新计数
2. 在`keepalive_timeout`时间内，若一直没有新的请求，一直处理空闲状态，则在超时后，nginx会主动断开该连接。**在发送第一个FIN包后，该连接将不能再被使用**

## Http Keepalive和Tcp Keepalive的差异
Http Keepalive是为了复用TCP连接，让其在空闲状态能够再存在一段时间，服务于后续的用户请求。Tcp Keepalive是检测TCP连接状态的一种保鲜机制。
### Tcp Keepalive的配置

``` bash
[root@VM_196_194_centos bearboyxu]# sysctl -a | grep keepalive
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_time = 7200
```

当TCP连接处于空闲状态时，当空闲时间超过`tcp_keepalive_time`后，服务器内核会尝试向client发送侦测包，来判断该链接的状态（可能存在客户端异常崩溃、网络异常等情况）。如果没有收到客户端的ACK确认报文，内核会在`tcp_keepalive_intvl`时间后再次尝试侦测。如果侦测`tcp_keepalive_probes`次后，依然没有收到client的响应，则会认为该连接已经不可用，内核选择丢弃该TCP连接。

> 操作系统默认的`tcp_keepalive_time`为2小时，一般生产环境会设置为30分钟

## 操作系统TCP参数的配置方法
### 内核查看方法
1. `uname -a`
2. `uname -r`
3. `cat /proc/version`

### TCP相关参数的配置文件
TCP相关的参数配置文件存放在系统目录`/proc/sys/net/ipv4` 和 `/proc/sys/net/ipv6` 中。查询Tcp Keepalive相关配置文件为

``` bash
[root@VM_196_194_centos bearboyxu]# ll /proc/sys/net/ipv4/tcp_keepalive_*
-rw-r--r-- 1 root root 0 4月   2 23:48 /proc/sys/net/ipv4/tcp_keepalive_intvl
-rw-r--r-- 1 root root 0 4月   2 23:48 /proc/sys/net/ipv4/tcp_keepalive_probes
-rw-r--r-- 1 root root 0 4月   2 23:48 /proc/sys/net/ipv4/tcp_keepalive_time
```

### 修改方法
一般使用`echo`命令直接修改配置文件内容

```bash
echo 60 > /proc/sys/net/ipv4/tcp_keepalive_time
```

### 查看配置
使用`sysctl -a`可以查看`/proc/sys`下的所有内容。从中可以过滤出`keepalive`相关的配置

```bash
[root@VM_196_194_centos bearboyxu]#  sysctl -a | grep keepalive
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_time = 7200
```

### 修改重启后生效
操作系统重启后，会从`/etc/sysctl.sys`中读取默认配置项，来初始化`/proc/sys/net/ipv4`目录下的对应配置文件。

> 在syscrl.conf中的修改需要重启后才能生效，如果想不重启生效需要运行`sysctl -p`命令。

如，通过`sysctl -w`命令设置keepalive相关配置,使用`sysctl \`进行配置查询：

```bash
[root@VM_196_194_centos bearboyxu]# sysctl -w \
> net.ipv4.tcp_keepalive_time=600 \
> net.ipv4.tcp_keepalive_intvl=60 \
> net.ipv4.tcp_keepalive_probes=20
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 20
[root@VM_196_194_centos bearboyxu]# sysctl -p
[root@VM_196_194_centos bearboyxu]# sysctl \
> net.ipv4.tcp_keepalive_time
net.ipv4.tcp_keepalive_time = 600

```
