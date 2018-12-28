---
layout: post
title: RedLock快速构造分布式锁
date: 2017-03-26 15:00:00
updated: 2017-03-26 15:00:00
description: 利用RedLock快速构造分布式锁，解决资源冲突和数据不一致等问题。对于构建分布式运行系统非常拥有重要的作用。
feature: images/tag-plugins/redis.png
comments: true
toc: true
tags:
- Python
- 分布式
- RedLock
categories:
- Python
---

在工作中往往会需要，对db中数据进行统计或者全量的数据处理。在数据量小、业务也不复杂的情况下，可以采用单进程执行。但是当数据量大、业务复杂、单次任务完成时间较长时，就需要采用多进程执行了。而对于一些更为复杂的场景，就需要采用"多机多进程"的方式来执行了。



>  在"分布式任务执行"中，最关键的便是保证任务的唯一。

<!--more-->



## 应用场景
 - 库内有1000W的数据，现在需要对每一条数据做具体的逻辑处理，该逻辑处理平均耗时为5S。（单进程完全执行需要115.7天）
 - 现在有100W的任务需要离线执行，每个任务都很耗CPU且耗时也很长，3个并发的进程几乎要把服务器CPU跑满，如果采用分布式执行，在高并发的情况下，如何保证每一个任务只会被执行一次呢？
 - 在分布式执行任务时，如果想要修改任务参数或者对任务队列中的任务做增、删、改的操作，如何保证执行器拿到的任务就是最新的呢？

对于一些数据量小，业务逻辑不复杂的任务，采用单机多进程的方法便能满足，只需要巧妙利用随机的思想，便能高概率的实现**任务的单例执行**

### 随机任务提取方法
执行sql查询时，利用`limit N`的方式从db中提取N个任务信息，在使用随机的方法从N个任务中随机地选取一个任务去执行。

``` python
[sql]   : select job from job_table where .... limit N 
		# NOTE: 从任务表job_table中提取满足条件的N个任务job_list
[Python]: sel_job = job_list[random.randint(0, len(job_list) - 1)]
          job_do(sel_job) # NOTE: job_do()拿到任务sel_job并执行
```


#### 性能瓶颈

该方法对于进程数较少的服务，还是能满足需求的，但是对于进程数太多，就会遇到瓶颈。

 - **可能出错的地方，就一定会出错。** 随机的方法虽然能够高概率的实现单例，但还是有可能会取到同一个任务，使得某一个任务被执行多次
 - 数较多的服务中，不仅会出现任务被多次执行的情况，还会对db造成不小的负担，如果sql查询不是采用索引的话，会全局扫描，不断的读写磁盘。并发请求越多，db的负担越重，等待时间就会更长。严重影响性能。

### 行锁提取方法
使用`select ... for update`的方式提取任务。行锁能保证一个任务只会被一个进程提起。

``` python
db_con = MySQLdb.connect(**DB_CONFIG)
db_cursor = db_con.cursor()
db_cursor.execute("set names utf8")
db_con.commit()

db_cursor.execute("begin")  # NOTE: 开始事物
try:
    # NOTE: 开始提取任务
    sql = """select job from job_table where ... order by ... limit 1 for update""" 
    result_cnt = db_cursor.execute(sql)

    if 0 == result_cnt:
        return None

    job = db_cursor.fetchone()
    # NOTE: 修改任务状态，为'正在执行中'
    sql = """update job_table set state = -1, modify_time = NOW() where job = %d""" % (job) 
    result_cnt = db_cursor.execute(sql)
    if 0 == db_con.affected_rows():
        return None
    return job
finally:
    db_con.commit() # NOTE: 提交事务
    db_cursor.close()
    db_con.close()
```



在查询语句后添加`for update`，DB在查询的过程中给Table添加排他锁。**InnoDB引擎在加锁的时候，只有通过索引进行检索的时候才会使用行级锁，否则会使用表级锁**

如果使用这个方法就必须使用索引进行查询，而且必须是 **唯一索引** 。



#### 性能瓶颈
 - where条件如果不走索引，极大可能会出现**锁表**的情况，严重影响多进程的任务提取性能（实测，很难受）
 - where条件只走索引，就只会进行**行锁**。但是被锁的行数跟匹配返回的行数可能不一致，这取决于查询在索引上匹配的行数（很坑，条件过滤掉的行也会被锁）
 - **MySql会对查询进行优化，是否使用索引来检索数据是由 MySQL 通过判断不同执行计划的代价来决定的，如果 MySQL 认为全表扫效率更高，比如对一些很小的表，它就不会使用索引，这种情况下 InnoDB 将使用表锁，而不是行锁。就很尴尬**



## **RedLock** 分布式锁



在分布式任务模型中，解决多进程互斥地访问资源的常有效的技术手段就是分布式锁。RedLock则是官方出的运用在分布式环境中的  <a href="https://redis.io/topics/distlock">redis分布式锁</a>





### 基于单实例的锁实现原理
对于一般的场景，使用单实例便可简单快速的实现分布式任务系统。`RedLock`的实现原理大致如下：
  - 客户端使用`SET resource_name my_random_value NX PX 30000`获得锁
      - resource_name 为锁名
      - NX 表明只有在resource_name不存在的时候才会设置这个key
      - PX 设置锁的超时时间。如果锁拥有者因为某一个逻辑卡住了，Redlock会在超时后，主动删除这个锁。
      - my_random_value 为一个随机值，该值必须与获得该锁的客户端保持一致，起到 **签名** 的作用。目的是为了防止误删，保证能够安全的释放锁，避免删除了其他客户端获得的锁。
      - 30000 key值得超时时间，也叫做"锁有效时间"。也就是一个客户端获得锁之后，处理逻辑的最大时间，超过这个时间，锁会被自动释放，然后被其他客户端抢占。

  - 对于`my_random_value`存在的意义
      - 客户端A拿到锁之后，被逻辑中某个操作阻赛了很长时间，最终超过了锁有效时间，锁被自动释放，该客户端失去该锁的拥有权

      - 客户端B采用同样的方法获得该锁，并在逻辑处理过程中

      - 客户端A试图删除该锁（逻辑执行完成），这时，如果没有*my_random_value*的签名证明，就会删除已经属于客户端B的锁，造成混乱。有了这个**签名**，客户端A通过它自己的*my_random_value*，是不可能删除客户端B的锁的。

        ​

### **RedLock算法**应用在多实例分布式系统中
多个完全独立的redis主站节点，客户端从N = 5个主站中获取锁的流程[RedLock算法][1]：
  - 获取当前时间t(ms)
  - 轮询的用(key, t)和随机值my_random_value在N个节点上请求锁。(客户端在每个master上请求锁时，会有一个和总的锁释放时间相比小的多的超时时间。比如如果锁自动释放时间是10秒钟，那每个节点锁请求的超时时间可能是5-50毫秒的范围，这个可以防止一个客户端在某个宕掉的master节点上阻塞过长时间，**如果一个master节点不可用了，我们应该尽快尝试下一个master节点。**)
  - 客户端计算第二步中获取锁所花的时间，只有当客户端在大多数master节点上成功获取了锁（在这里是3个），而且总共消耗的时间不超过锁释放时间，这个锁就认为是获取成功了。
  - 如果锁获取成功了，那现在**锁自动释放时间就是最初的锁释放时间减去之前获取锁所消耗的时间**。
  - 如果锁获取失败了，不管是因为获取成功的锁不超过一半（N/2+1)还是因为总消耗时间超过了锁释放时间，客户端都会到每个master节点上释放锁，即便是那些他认为没有获取成功的锁。

    ​

### **RedLock** 分布式锁的应用  
redis提供了很多语言版本的demo，这里根据业务需要，使用python来搭建分布式任务系统。[官网demo][2]



#### **RedLock** 分布式锁的实现
基于`redlock`构造RedLock锁类。这里的设计思路来源于同事的一个爬虫框架，`with`的使用很巧，就搬过来拿到现在的项目中使用了。
业务逻辑大致可以分为三步
  - 1 执行逻辑之前，获取RedLock

  - 2 执行具体逻辑

  - 3 逻辑执行完成之后，释放RedLock

    ​

**这里的流程特点，刚好可以利用`with`的特性来设计完成：**

``` python
# -*- coding: utf-8 -*-
from __future__ import (absolute_import, unicode_literals)
import redlock
import time

class distributed_locking(object):
    def __init__(self, **config):
        self.config = config
        self.dlm = redlock.Redlock(
                                [config['server'], ], # the configs of all master
                                retry_count = config['retry_count'],
                                retry_delay = config['retry_delay'])
        self.dlm_lock = None

    def __enter__(self): #使用with语句时调用，会话管理器在代码块开始前调用
        while not self.dlm_lock:
            # Where the resource name is an unique identifier of what you are trying to lock
            # and 1000 is the number of milliseconds for the validity time.
            self.dlm_lock = self.dlm.lock(self.config['resource_name'], 1000)
            if self.dlm_lock: # acquire lock
                break
            else:
                time.sleep(self.config['retry_delay']) # wait for lock

    def __exit__(self, type, value, traceback): #会话管理器在代码块执行完成好后调用（不同于__del__）
        self.dlm.unlock(self.dlm_lock) # release lock
        self.dlm_lock = None

    def __del__(self):  
        print "__del__"
```
#### 客户端利用 `distributed_locking()` 完成任务的提取

``` python

DISTRIBUTED_LOCK_CONFIG = {
    'server': {
        'host': 'localhost',
        'port': 6379,
        'password': None,
        'db': 1,
    },
    'resource_name': 'distributed_lock_name',
    'retry_count': 5,
    'retry_delay': 0.2,
}

# 利用`with`的`__init__`、`__enter__`和`__exit__`完成锁的初始化、获取锁、释放锁操作。
with distributed_locking(**DISTRIBUTED_LOCK_CONFIG):
    # 业务逻辑
```

<p></p>

业务层通过获取redis分布式锁的方式，解决了进程间互斥的问题，还很好的保证了分布式系统中任务的唯一性。

就目前上线的系统来看，还是比较稳定的。

当然，这样的设计并不是完美的，但是对于一般的业务，能够快速搭建分布式任务系统


[0]: https://redis.io/topics/distlock
[1]: http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
[2]: https://github.com/SPSCommerce/redlock-py
