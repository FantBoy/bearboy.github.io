---
layout: post
title: 【Exception】 MISCONF Redis is configured to save RDB snapshots
date: 2017-04-19 20:35:11.000000000 +09:00
updated: 2017-04-19 20:35:11.000000000 +09:00
comments: true
tags:
- 异常记录
categories:
- Redis
---

今天突然redis服务出现了异常，上机器看log，提示说"MISCONF Redis is configured to save RDB snapshots"
<!--more-->

## 异常详情

``` shell
Traceback (most recent call last):
  File "job_story_script.py", line 146, in <module>
    main(0)
  File "job_story_script.py", line 142, in main
    add_jobs()
  File "job_story_script.py", line 106, in add_jobs
    red.lpush('t_programe_download_nomal', programe_id)
  File "/usr/local/services/python2-1.0/lib/python2.7/site-packages/redis/client.py", line 1210, in lpush
    return self.execute_command('LPUSH', name, *values)
  File "/usr/local/services/python2-1.0/lib/python2.7/site-packages/redis/client.py", line 565, in execute_command
    return self.parse_response(connection, command_name, **options)
  File "/usr/local/services/python2-1.0/lib/python2.7/site-packages/redis/client.py", line 577, in parse_response
    response = connection.read_response()
  File "/usr/local/services/python2-1.0/lib/python2.7/site-packages/redis/connection.py", line 574, in read_response
    raise response
redis.exceptions.ResponseError: MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk. Commands that may modify the data set are disabled. Please check Redis logs for details about the error.
```


第一反应是不是机器的磁盘写满了，无法本地持久化，可看磁盘信息，发现还有好大的空间~~
``` shell
[# /usr/local/services/create_programes_peaks_server-1.0/log]$ df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/sda1             9.9G  3.4G  6.0G  37% /
/dev/sda3              20G   19G     0 100% /usr/local
/dev/sda4             1.5T  2.9G  1.4T   1% /data
tmpfs                  16G   24K   16G   1% /dev/shm
```

登陆redis-server，尝试lpush一条新纪录，返回同样的异常

``` shell
127.0.0.1:6379[1]> lpush t_programe_download_autosync_new_cfs 1234
(error) MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk. Commands that may modify the data set are disabled. Please check Redis logs for details about the error.
```

> 异常含义：Redis被配置为保存数据库快照，但它目前不能持久化到硬盘。用来修改集合数据的命令不能用。请查看Redis日志的详细错误信息

## 原因

强制关闭Redis快照的操作导致不能持久化。**至于为什么会出发这个问题，还没查出来**

## 解决方案
将stop-writes-on-bgsave-error设置为no
``` shell
127.0.0.1:6379> config set stop-writes-on-bgsave-error no
```
