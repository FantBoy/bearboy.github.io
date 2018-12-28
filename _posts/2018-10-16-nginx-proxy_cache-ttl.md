---
layout: post
filename: nginx-proxy_cache-ttl
title: nginx之proxy_cache过期管理
datetime: 2018-10-16 00:09:13
description: nginx之proxy_cache过期管理
comments: true
tags:
 Nginx
categories:
 Nginx
 
---

## 原理

proxy_cache缓存过期涉及三个变量：expired、inactive、proxy_cache_valid



其中：

- expired：缓存数据的过期时间，过期后不会删除数据

- inactive：冷数据删除时间。如果缓存数据在inactive时间内没有被访问，则删除（默认值10min）

  >Cached data that are not accessed during the time specified by the **inactive** parameter get removed from the cache regardless of their freshness. By default,**inactive** is set to 10 minutes.

- proxy_cache_valid：缓存的最大可用时间。在保证inactive内有访问的前提下，数据的过期时间
  >Sets caching time for different response codes. For example, the following directives
**proxy_cache_valid 200 302 10m;
proxy_cache_valid 404      1m;**
set 10 minutes of caching for responses with codes 200 and 302, and 1 minute for responses with code 404.
**expires**: Controls whether the response should be marked with an expiry time, and if so, what time that is.



inactive时间表示一个资源数据在该时间内没有被访问过的话，就会从缓存中移除，而不用关注proxy_cache_valid。proxy_cache_valid在保证inactive时间内数据被访问过的前提下，数据的最长可用时间（过期时间）。若数据过期，则会重新获取资源，即使资源被高频率访问。



## 基本配置项

### proxy_cache_path配置

```
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m use_temp_path=off;
server {
    
}
```

配置项说明：

- /path/to/cache : 本地路径，缓存文件存放地址
- levels : 默认所有缓存文件都放在同一个/path/to/cache下，从而影响缓存的性能，大部分场景推荐使用2级目录来存储缓存文件（`levels=1:2`表示两级目录）
- key_zone : 在共享内存中设置一块存储区域来存放缓存的key和metadata（类似使用次数），这样nginx可以快速判断一个request是否命中或者未命中缓存，1m可以存储8000个key，10m可以存储80000个key
- max_size : 最大cache空间，如果不指定，会使用掉所有disk space，**当达到配额后，会删除缓存时间最长的资源**
- inactive : 未被访问文件在缓存中保留时间，本配置中如果60分钟未被访问则不论状态是否为expired，缓存控制程序会删掉文件，默认为10分钟；**inactive和expired配置项的含义是不同的，expired只是缓存过期，但不会被删除，inactive是删除指定时间内未被访问的缓存文件**
- use_temp_path : 如果为off，则nginx会将缓存文件直接写入指定的cache文件中，而不是使用temp_path存储，official建议为off，避免文件在不同文件系统中不必要的拷贝
- proxy_cache : 启用proxy cache，指定key_zone



### proxy_cache_valid配置

- 全局唯一设置缓存时间:  `proxy_cache_valid 5m;`

- 对指定响应码设置缓存时间：

  ```
  proxy_cache_valid 200 302 10m; 
  proxy_cache_valid 301 1h; 
  proxy_cache_valid 404 1m; 
  proxy_cache_valid any 1m;
  ```

### 响应header中的设置

- `X-Accel-Expires`，设置响应的缓存过期时间（秒），0表示不缓存
- 如果header含有`Set-Cookie`，则响应不会被缓存








