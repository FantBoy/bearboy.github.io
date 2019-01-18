---
layout: post
filename: ngx_http_limit_req_and_conn_module
title: Nginx配置之用户请求限频、流量控制和白名单配置
datetime: 2019-01-16 23:07:32
description: Nginx配置之用户请求限频和流量控制
comments: true
tags:
 - Nginx
categories:
 - Nginx
 
---



在Nginx的高并发场景，往往需要对用户请求进行限频或者限流。以此来保护后台服务的稳定运行。Nginx已有的两个模块为限频和限流提供了支持：`ngx_http_limit_req_module` 请求限制模块和`ngx_http_limit_conn_module` 流量限制模块。同时，提供黑白名单的配置能力，根据用户的ip和黑名单列表做出相应的流量控制和请求限频。



## 按照请求连接数进行流量控制

Nginx提供了`ngx_http_limit_conn_module`模块，根据请求连接频率来进行流量控制。配置demo:

![](http://blog.bearboyxu.cn/images/posts/ngx_http_limit_module/limit_conn_zone.png)



配置描述：

| 配置项               | 配置描述                                                     |
| :------------------- | :----------------------------------------------------------- |
| limit_conn_zone      | 用来限流的KEY，及存放KEY对应信息的共享内存区域大小。KEY为`$binary_remote_addr`时，表示按照IP维度限制最大连接数；KEY为`$server_name`时，表示按照域名维度限制最大连接数。当zone设定的共享内存空间被耗尽后，Nginx服务器直接向用户返回503（默认返回码） |
| limit_conn           | 对某个KEY对应的总的网络连接数进行限流，可以按照IP，也可以按照域名来做限制。当用户请求被Nginx处理过，且已经读取了整个用户请求的请求头，这个请求将会被计数。 |
| limit_conn_log_level | 当请求被限流后的Nginx日志级别，默认为`error`级别             |
| limit_conn_status    | 当请求被限流后，Nginx返回的相应状态码（默认为503）           |

`limit_conn zone number`指定一块已经被设定的内存空间，以及每个键值（KEY）的最大连接数。当用户真实连接数超过了最大连接数number，Nginx直接向用户返回503（默认返回码）。



> 当多个`limit_conn`在同一个配置块被配置时，所有的配置都生效



## 按照请求数（频率）限流
