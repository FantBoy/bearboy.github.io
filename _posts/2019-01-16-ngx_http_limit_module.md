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

配置样例：

```nginx
limit_req_zone $binary_remote_addr zone=perip:10m rate=1r/s;
limit_req_zone $server_name zone=perserver:10m rate=10r/s;

server {
    ...
    limit_req zone=perip burst=5 nodelay;
    limit_req zone=perserver burst=10;
}
```



### limit_req_zone限流基本配置

```nginx
limit_req_zone [key] zone=[name]:[size] rate=[rate]
```

| 配置 | 配置项说明                                                   |
| :-------- | :------------------------------------------------------------ |
|    key    | 表示限制的关键字，可以是用户IP`$binary_remote_addr`,也可以是`$server_name`虚拟服务域名 |
|    zone   | name可以自定义，但是不能重复。它代表一个存储session状态的容器；size表示容器大小（按照64-byte一个session来计算，一共可以存储size/64-byte个session） |
|    rate   | 表示请求的频率限制。单位为`r/s`表示每秒的请求频率限制；`r/m`表示每分钟的请求频率限制 |

```nginx
#每个用户IP的请求频率，每秒不能超过1次，且最大存储容量为10M
limit_req_zone $binary_remote_addr zone=perip:10m rate=1r/s;

#每个虚拟服务域名的请求每秒不能超过10次，且最大存储容量为10M
limit_req_zone $server_name zone=perserver:10m rate=10r/s;
```

> 当Nginx需要新增记录时，如果存储空间不足，Nginx将会删除旧的记录（为了防止内存被耗尽，每次创建新的记录时，最多删除两条60秒内未使用的记录）。如果释放掉的空间仍然不足以容纳新的记录，Nginx会直接向用户返回503。



### limit_req 限流具体策略

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



### 日志配置

```nginx
limit_req_log_level info|notice|warn|error
```

默认情况下，Nginx会记录由于流量限制而被延迟或丢弃的请求日志。Nginx默认以error级别记录被丢弃的请求，以较低级别记录被延迟的请求（默认情况为info级别）。

> 当服务器因为频率过高拒绝或者延迟处理请求时，日志记录时的日志级别。延迟记录的日志级别比拒绝服务的日志级别低一个级别。



日志格式：

- limiting requests： 表明该日志记录为请求被“流量限制”
- excess： 每毫秒超过对应“流量限制”配置的请求数量
- zone：定义实施“流量限制”的区域
- client：发起请求的客户端ip地址
- server：服务器ip地址或主机名
- request：客户端发起的实际HTTP请求
- host：HTTP请求报文中的host头域



## 如何处理大量的突发请求

nginx处理大量突发请求，取决于`burst`配置。burst定义了请求数在超过`zone`定义的请求速率的情况下，服务器将超过部分请求缓存，延迟处理。最大的请求缓存数极为`burst`值。当缓存的请求数超过了`burst`值，则不延迟，直接向用户返回503错误码（默认503）。

以下配置为例：

```nginx
limit_req_zone key zone=mylimit:10m rate=10r/s
limit_req zone=mylimit burst=20
```

如果用户在100ms内并发了21条请求，Nginx会将第一个请求直接发送给上游服务处理，将后续的20个请求做缓存处理，放入队列中，然后每100ms（根据`rate`计算的最小处理周期）处理一个请求。当有新请求进来时，如果队列已经装满，则直接返回503（默认）。

### 对缓存的请求做不延迟的处理

如上面所描述，配置`burst`参数，将会使用户请求更流畅，能够有效的应对突然的请求激增。但是超过请求频率的请求会被延迟处理，从一定程度上增加了用户请求响应时延。

`ngx_http_limit_req_module`模块提供了`nodelay`配置项来优化这一问题

使用了`nodelay`，Nginx将会根据`burst`参数分配队列中的位置。应用已配置的速率来限制。

```nginx
limit_req zone=[name] burst=[count] nodelay
```

当一个请求到达时（已经超过了请求频率限制的场景），只要队列中还有空闲的位置可以分配。Nginx会立即处理该请求，同时在队列中为该请求分配一个位置，并置为`taken（占据）`。被占据的位置不会被释放给其他请求，直到一段时间被释放后才能被其他请求复用（根据`rate=10r/s`计算释放时间为100ms）。

> 大部分场景下，建议将`burst`和`nodelay`配置项结合使用。

## 不限流白名单配置

Nginx提供`gro`指令配置针对用户请求的黑白名单配置。demo如下所示：

```nginx
geo $whiteIpList {
    default         1;
    192.168.5.0/24  0;
}
 
map $whiteIpList $limit_key {
    1 $binary_remote_addr;
    0 "";
}
 
limit_req_zone $limit_key zone=mylimit:10m rate=10r/s;
 
location / {
        limit_req zone=mylimit burst=20 nodelay;
        ...
}
```

技术要点：

1. `geo`指令定义了一个白名单列表，默认值都为1，所有ip都受限制。如果客户端ip与白名单列出的ip相匹配，则`$whiteIpList`中值为0的ip将不受限制。
2. `map`指令是将`$whiteIpList`中值为1的ip，也就是受限制的ip设置为客户端ip。将`$whiteIpList`中值为0的ip，也就是白名单ip映射为空字符串。
3. `limit_conn_zone`和`limit_req_zone`指令对于KEY为空字符串的请求，将会被忽略，不做任何限制，从而实现对于白名单中设置的白名单ip不做流量限制。



## 指定location拒绝所有请求

在location中添加`deny all`指令即可实现，对指定location的请求，全部拒绝。

