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

Nginx提供了`ngx_http_limit_conn_module`模块，根据请求连接频率来进行流量控制。相关配置说明:
```nginx
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_conn_zone $server_name zone=perserver:10m;

server {
    ...
    limit_conn perip 10;
    limit_conn perserver 100;
}
```

> 当多个`limit_conn`在同一个配置块被配置时，所有的配置都生效




| 配置项               | 配置描述                                                     |
| :------------------- | :----------------------------------------------------------- |
| limit_conn_zone      | 用来限流的KEY，及存放KEY对应信息的共享内存区域大小。KEY为`$binary_remote_addr`时，表示按照IP维度限制最大连接数；KEY为`$server_name`时，表示按照域名维度限制最大连接数。当zone设定的共享内存空间被耗尽后，Nginx服务器直接向用户返回503（默认返回码） |
| limit_conn           | 对某个KEY对应的总的网络连接数进行限流，可以按照IP，也可以按照域名来做限制。当用户请求被Nginx处理过，且已经读取了整个用户请求的请求头，这个请求将会被计数。 |
| limit_conn_log_level | 当请求被限流后的Nginx日志级别，默认为`error`级别             |
| limit_conn_status    | 当请求被限流后，Nginx返回的相应状态码（默认为503）           |

`limit_conn zone number`指定一块已经被设定的内存空间，以及每个键值（KEY）的最大连接数。当用户真实连接数超过了最大连接数number，Nginx直接向用户返回503（默认返回码）。





## 按照请求数（频率）限流

Nginx提供了`ngx_http_limit_req_module`模块，根据单个用户IP或者域名的请求频率来进行流量控制。

### 相关配置

```nginx
limit_req_zone [key] zone=[name]:[size] rate=[rate]
```

| 配置项 | 配置项说明                                                   |
| ------ | ------------------------------------------------------------ |
| key    | 表示限制的关键字，可以是用户IP`$binary_remote_addr`,也可以是`$server_name`虚拟服务域名 |
| zone   | name可以自定义，但是不能重复。它代表一个存储session状态的容器；size表示容器大小（按照64-byte一个session来计算，一共可以存储size/64-byte个session） |
| rate   | 表示请求的频率限制。单位为`r/s`表示每秒的请求频率限制；`r/m`表示每分钟的请求频率限制 |

```nginx
#每个用户IP的请求频率，每秒不能超过1次，且最大存储容量为10M
limit_req_zone $binary_remote_addr zone=perip:10m rate=1r/s;

#每个虚拟服务域名的请求每秒不能超过10次，且最大存储容量为10M
limit_req_zone $server_name zone=perserver:10m rate=10r/s;
```

> 当Nginx需要新增记录时，如果存储空间不足，Nginx将会删除旧的记录（为了防止内存被耗尽，每次创建新的记录时，最多删除两条60秒内未使用的记录）。如果释放掉的空间仍然不足以容纳新的记录，Nginx会直接向用户返回503。



```nginx
limit_req zone=[name] burst=[count] nodelay
```

`limit_req`配置可以放在server中，也可以放在location中，放在server中时表示对整个服务做限流；放在location中表示只对特定的请求做限流。

| 配置项  | 配置项说明                                                   |
| :------ | :----------------------------------------------------------- |
| zone    | 选择限流容器，`name`为`limit_req_zone`中设定的限流容器名称   |
| burst   | 可缺省，默认值为0。超过频率限制部分，被缓存的请求数量，`count`为最大缓存请求数 |
| nodelay | 可缺省。表示不延迟，即如果请求数超过频率限制时，不延迟缓存   |

设置对应的共享内存限制域和允许被处理的最大请求数值域。如果请求的频率超过了请求频率限制`rate`，请求的处理将会被延迟。直到被延迟的请求数超过了最大缓存请求数`count`，用户	请求将会终止，直接返回503（默认返回码）。



> `limit_req_zone`指令设置流量频率控制和共享内存区域参数。但是真正做流量限制的时`limit_req`。其应用在特定的location/server块。



```nginx
limit_req_log_level info|notice|warn|error
```

限流时的日志级别，默认为error级别。

当服务器因为频率过高拒绝或者延迟处理请求时，日志记录时的日志级别。延迟记录的日志级别比拒绝服务的日志级别低一个级别。





```nginx
limit_req_zone $binary_remote_addr zone=perip:10m rate=1r/s;
limit_req_zone $server_name zone=perserver:10m rate=10r/s;

server {
    ...
    limit_req zone=perip burst=5 nodelay;
    limit_req zone=perserver burst=10;
}
```

