---
layout: post
title: Nginx热升级方案
datetime: 2019-01-29 22:21:32
description: Nginx热升级方案
comments: true
tags:
 - Nginx
categories:
 - Nginx
---

在Nginx的正常运行中，如果更新了配置，往往可以通过reload的方式来实现更新：使用新的配置启动新的工作进程，并逐步关闭久的工作进程，平滑实现新老配置的切换。但是当Nginx二进制文件或增删改模块时，reload将无效。为了在升级过程中不丢失用户请求，又不能对Nginx服务进行重启操作。好在Nginx提供了一系列的信号可以辅助我们完成**热升级**。USR2、WINCH和QUIT等信号可以很好的实现Nginx的平滑升级，达到热升级的目的，同时还可以做到回滚操作。
<!--more-->

## Nginx在多进程模式下的工作模式
说到Nginx的热升级，有必要先补充一下Nginx在多进程模式下的工作模式（master-workers）。
Nginx默认工作在多进程模式下，即主进程（master process）启动后完成了相关配置的加载和端口绑定工作，并fork出指定数量的工作进程（work process）。这些工作进程会持有监听端口的文件描述符（fd）,并通过在该fd上添加监听事件来接受连接（accept）和处理请求。具体实现可以查看[ngx_process_cycle.c](https://github.com/nginx/nginx/blob/master/src/os/unix/ngx_process_cycle.c)。

> Nginx在主进程启动完成后，便进入等待状态，负责响应各类系统消息，如SIGCHLD、SIGHUP、SIGUSR2等信号。

## Nginx支持的信号简介
### 主进程支持的信号
 - TERM，INT：立刻退出，强制关闭整个服务
 - QUIT：等待工作进程结束后退出，优雅地关闭整个服务
 - KILL：强制终止进程
 - HUP；重新加载配置，使用新配置启动新的工作进程，并逐步关闭旧的工作进程，使服务对新配景项生效  
 - USR1：重新打开股务中的所有文件
 - USR2：启动新的主进程`Nginx的热更新主要依赖该信号`
 - WINCH：逐步关闭工作进程

### 工作进程支持的信号
 - TERM，INT：立刻退出
 - QUIT：等待工作进程处理完当前请求后退出
 - USR1：重新打开股务中的所有文件

### nginx -s signal支持的信号
 - stop：等价于TERM，INT
 - quit：QUIT
 - reopen：USR1
 - reload：HUP


## reload的工作流程

一般的配置项变更可以通过`nginx -p .. -s reload`完成动态加载配置。具体步骤如下：
 - 改变Nginx配置后，执行`reload`命令，发送`HUP signal`的信号给Master主进程
 - Master主进程接收到信号后，检查新的配置的语法有效性
 - 尝试使用新配置
    1. 打开日志文件，新配一个socket来监听。
    2. 第1步若成功，则使用新的配置，新建worker工作进程。新建成功后发送一个`QUIT`给旧的worker进程，要求旧的进程优雅的退出。
    3. 第一步若失败，则回滚操作，还是使用旧的工作进程。
    4. 旧的进程在收到`QUIT`信号时，会继续服务于正在处理的请求，当所有请求在旧进程都被处理完成后，旧进程优雅的关闭。


```shell
kill -HUP $(cat logs/nginx.pid)
```


## 热升级步骤
当Nginx二进制文件或增删改模块时，需要进行热升级操作，因为reload已经不能满足该场景。
1. 备份二进制文件：nginx和*.so。
2. 编译新版本的nginx和so。
3. 向Master主进程发送`USR2`信号。Nginx主进程Master Process会以新配置、新文件启动新的主进程和工作进程组。新老进程组一起为用户提供服务。
   
   ![执行热升级前](/images/posts/nginx_hot_update/before_usr2.png)
   
   ![执行USR2后](/images/posts/nginx_hot_update/end_usr2.png)
4. 向原Master主进程发送`WINCH`信号，它会让原Master主进程逐步关闭原工作进程组（请求处理完成后优雅退出）。这时，新的用户请求全部交由新的工作进程服务（老的Master主进程还存在）。
   
   ![执行WINCH后](/images/posts/nginx_hot_update/winch.png)

5. 检查新的工作进程的工作状态，查看服务服务是否OK。若新工作进程服务OK，则通过`KILL`将老的Master主进程杀死，完成热升级。若需要回滚操作，则向原Master主进程发送`HUP`信号，它会使用旧版配置重新拉起工作进程。然后再将新的Master主进程和Worker工作进程组杀死（QUIT或KILL），实现回滚。
   ![kill旧的主进程](/images/posts/nginx_hot_update/kill.png)
> 在切换过程当中，Nginx会将旧的`.pid`文件重命令为`.pid.oldbin`文件。并在旧进程退出后删除。