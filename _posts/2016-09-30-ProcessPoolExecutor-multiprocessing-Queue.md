---
layout: post
title: 基于python的多进程并行处理模型
date: 2016-09-30 13:34:11.000000000 +09:00
description: 利用concurrent实现简单的多进程并发,处理平时工作中常需要的大批量离线处理任务
feature: images/tag-plugins/python.jpg
comments: true
toc: true
tags:
- Python
- Multiprocessing
categories:
- Python
---



在工作中，常常会遇到需要离线处理大批量的数据或者任务的需求。例如现在有2000W的数据需要离线处理，且每个数据的逻辑处理都会消耗一定的时间，如果采用单进程处理，往往需要消耗大量的时间，少则一两个小时，多则一天（同事可不会等你慢慢的去执行）。利用concurrent模块开发多进程并发处理模型，能够快速的为我们解决类似问题。使用简单，拿来即用。该模型主要应用了：


- **concurrent.futures**  - 在`pthon3`中是内嵌的库，`python2.x`需要另行安装 。主要用于多线程 & 多进程的并发处理
- **Executor** - 提供两个子类分别创建线程池和进程池：`ThreadPoolExecutor`，`ProcessPoolExecutor`
- **Future** -  由`Executor.submit`产生多个任务
<!--more-->



### concurrent.future使用实例 ##

在python2.7中使用concurrent.futures的实例文档  - <a href="http://www.dalkescientific.com/writings/diary/archive/2012/01/19/concurrent.futures.html">Python's concurrent.futures </a>

the motivation for the new concurrent module  - <a href="https://www.python.org/dev/peps/pep-3148/">futures - execute computations asynchronously</a>





#### THreadPoolExecutor ##

``` python
with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
    print(list(executor.map(sleeper, x)))
```



#### ProcessPoolExecutor #####

`ProcessPoolExecutor`的用法相当的简练，不像以前的threading、multiprocessing那样繁琐


``` python
from concurrent.futures import ProcessPoolExecutor
def pool_factorizer_go(nums, nprocs):
   nprocs=xxx
    with ProcessPoolExecutor(max_workers=nprocs) as executor:
        return {num:factors for num, factors in
                                zip(nums,
                                    executor.map(factorize_naive, nums))}
```



### 简单的多进程并行处理模型 ###

#### 构造`Job`任务类，作为进程执行的最小单元 ####
`Job`：包含任务对象（函数）、任务id、args参数、kwargs参数、其他


``` python
class Job():
    def __init__(self, kwargs):
        self.id = kwargs.get('id', None)
        self.func = kwargs.get('func', None)
        self.args = kwargs.get('args', None)
        self.kwargs = kwargs.get('kwargs', None)
```



#### 构造进程池对象 ####

`concurrent.futures.ProcessPoolExecutor`： 构造进程池对象，包含进程池初始器和任务执行器

 - 通过`submit`提交`Job`任务给进程池，并附带参数
 - 通过`future.add_done_callback`设置回调函数，处理`Job`运行结果


``` python
class ProcessPoolDemo():
    def __init__(self, max_size):
        self._pool = concurrent.futures.ProcessPoolExecutor(max_workers=max_size)

    def excutor_job(self, job):
        def job_executed(f):
            if f.exception():
                print 'job[{job_id}] worker[{fun}] error : [{error_msg}]'.format(
                  job_id = job.id, fun = job.func, error_msg = f.exception_info())
            else:
                print 'job[{job_id}] worker[{fun}] finish: [{msg}]'.format(
                  job_id = job.id, fun = job.func, msg = f.result())
        future = self._pool.submit(job.func, job.args, job.kwargs)
        future.add_done_callback(job_executed)
```



#### multiprocessing.Queue实现简单的进程锁 ####

利用`Queue`的特点实现进程锁，不至于在多进程异步执行时出现紊乱

 - **特点** 先进先出、后进后出
 - `put`方法：当队列已经装满时，以阻塞的方式等待，直到有空间后再`put`
 - `get`方法：当队列为空时，以阻塞的方式等待，直到队列中有内容之后，在`get`

``` python
from multiprocessing import Queue
self.internal_job_queue = Queue(maxsize=30 * Max_workers)
self.internal_job_queue.put("#", True) #阻塞结束后，再执行job1
｛job1｝

self.internal_job_queue.get(False) #当队列中有元素，取出后，再执行job2
｛job2｝
```


#### 完整实例 ####

``` python
#!/usr/local/services/python2-1.0/bin/python2.0
# --*-- coding:utf-8 --*--
from __future__ import (absolute_import, unicode_literals)
import time
import random
import concurrent.futures
from multiprocessing import Queue


class Job():
    def __init__(self, kwargs):
        self.id = kwargs.get('id', None)
        self.func = kwargs.get('func', None)
        self.args = kwargs.get('args', None)
        self.kwargs = kwargs.get('kwargs', None)

class ProcessPoolDemo():
    def __init__(self, max_size):
        self._pool = concurrent.futures.ProcessPoolExecutor(max_workers=max_size)
        self.internal_job_queue = Queue(maxsize = max_size) #与进程数一致

    def excutor_job(self, job):
        def job_executed(f):
            self.internal_job_queue.get(False) #释放一个进程资源
            if f.exception():
                print 'job[{job_id}] worker[{fun}] error : [{error_msg}]'.format(
                    job_id = job.id, fun = job.func, error_msg = f.exception_info())
            else:
                print 'job[{job_id}] worker[{fun}] finish: [{msg}]'.format(
                    job_id = job.id, fun = job.func, msg = f.result())
        future = self._pool.submit(job.func, job.args, job.kwargs)
        future.add_done_callback(job_executed)

def fun_demo(args, kwargs):
    sleep_time = random.uniform(0, 1)
    time.sleep(sleep_time)
    return sleep_time

def main(args):
    process_pool = ProcessPoolDemo(4) #开启四个进程

    for index in range(0,10): #异步处理十个任务
        process_pool.internal_job_queue.put('#', True) # 一个进程
        print 'put in job[{job_id}]'.format(job_id = index)
        job_dict = {'id': index, 'func' : fun_demo, 'args' : [1,2,3], 'kwargs' : {'name' : 'bear', 'age': 26}}
        process_pool.excutor_job(Job(job_dict))

if __name__ == '__main__':
    main(0)
```

#### Output
```
put in job[0]
put in job[1]
put in job[2]
put in job[3]
put in job[4]
job[2] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.0703135307709]
job[4] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.128978704821]
put in job[5]
job[0] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.309983499388]
put in job[6]
job[5] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.186503637742]
put in job[7]
job[6] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.162506532634]
put in job[8]
job[1] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.62530039205]
put in job[9]
job[3] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.707108629808]
job[7] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.386662916358]
job[8] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.361320846488]
job[9] worker[<function fun_demo at 0x00000000029A7828>] finish: [0.325288205939]
```
