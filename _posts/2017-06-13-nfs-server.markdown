---
layout: post
title: 基于NFS部署分布式文件系统
date: '2017-06-14 19:17'
tags:
  - NFS
  - RPC
categories:
  - 分布式文件系统
---

> NFS 是Network File System的缩写，即网络文件系统。一种使用于分散式文件系统的协定。NFS 的基本原则是“容许不同的客户端及服务端通过一组RPC分享相同的文件系统”，它是独立于操作系统，容许不同硬件及操作系统的系统共同进行文件的分享，它的文件传送或信息传送过程依赖于RPC协议。

<!--more-->
# NFS服务安装

nfs服务通过yum执行安装
```shell
[# /home/user_00]# yum -y install nfs-utils
Loaded plugins: fastestmirror
Setting up Install Process
Determining fastest mirrors
 * base: 127.0.0.1
 * cr: 127.0.0.1
 * epel: 127.0.0.1
 * fasttrack: 127.0.0.1
 * tlinux:127.0.0.1
 * tlinux-SRPMS: 127.0.0.1
 * tlinux-debuginfo: 127.0.0.1
 * tlinux-tkernel2: 127.0.0.1
 * updates: 127.0.0.1
Package 1:nfs-utils-1.2.3-16.tl1.x86_64 already installed and latest version
Nothing to do
```

nfs-utils在安装的同时，rpcbind程序也会随着安装
**linux系统基本上都会默认安装nfs-utils服务和rpcbind程序，两者的安装包在系统的安装盘里都能找到**

## 安装检测

```shell
[# /home/user_00]# rpm -qa |grep nfs
nfs-utils-lib-1.1.5-4.el6.x86_64
nfs-utils-1.2.3-16.tl1.x86_64
[# /home/user_00]# rpm -qa | grep rpcbind
rpcbind-0.2.0-8.el6.x86_64
```

# NFS服务器

## NFS服务器系统的守护进程

 - nfsd：它是基本的NFS守护进程，主要功能是管理客户端是否能够登录服务器；
 - mountd：它是RPC安装守护进程，主要功能是管理NFS的文件系统。当客户端顺利通过nfsd登录NFS服务器后，在使用NFS服务所提供的文件前，还必须通过文件使用权限的验证。它会读取NFS的配置文件/etc/exports来对比客户端权限。
 - portmap：主要功能是进行端口映射工作。当客户端尝试连接并使用RPC服务器提供的服务（如NFS服务）时，portmap会将所管理的与服务对应的端口提供给客户端，从而使客户可以通过该端口向服务器请求服务。

## 服务器配置

NFS服务器根据不同的功能区分了不同的配置文件，相对简单

### 配置文件目录

NFS服务器配置的常用目录：

| 配置文件目录              |        配置文件说明        |
| :------------------ | :------------------: |
| /etc/exports        |     NFS服务的主要配置文件     |
| /usr/sbin/exportfs  |      NFS服务的管理命令      |
| /usr/sbin/showmount |       客户端的查看命令       |
| /var/lib/nfs/etab   | 记录NFS分享出来的目录的完整权限设定值 |
| /var/lib/nfs/xtab   |    记录曾经登录过的客户端信息     |

其中，`/etc/exports` 是NFS服务的主要配置文件，该配置文件在系统中不存在默认值，所以该文件可能并不存在，在第一次启动服务之前，需要手动创建。

### exports配置文件的文件内容格式

```Shell
<输出目录> [客户端1 选项（访问权限,用户映射,其他）] [客户端2 选项（访问权限,用户映射,其他）]
```

其中：
 - 输出目录： NFS系统中，服务器端共享给客户端使用的共享目录
 - 客户端： 网络中可以访问服务器共享目录的客户端机器，客户端机器可以为
  - 指定ip地址的主机： 192,168.1.123
  - 指定子网的所有主机： 192.168.0.0/24 192.168.0.0/255.255.255.0
  - 指定域名的主机： david.bsmart.cn
  - 指定一级域中的所有主机：\*.bsmart.cn
  - 所有主机：\*
 - 选项： 设置共享目录的用户访问权限和用户映射关系

各类选项说明：

| 访问权限选项 |    权限说明    |
| :----: | :--------: |
|   ro   | 设置输出目录只读权限 |
|   rw   | 设置输出目录读写权限 |


|     用户映射选项     |                   关系说明                   |
| :------------: | :--------------------------------------: |
|   all_squash   | 将远程访问的所有普通用户及所属组都映射为匿名用户或用户组（nfsnobody）  |
| no_all_squash  |           与all_squash取反（默认设置）            |
|  root_squash   |      将root用户及所属组都映射为匿名用户或用户组（默认设置）       |
| no_root_squash |              与rootsquash取反               |
|  anonuid=xxx   | 将远程访问的所有用户都映射为匿名用户，并指定该用户为本地用户（UID=xxx）  |
|  anongid=xxx   | 将远程访问的所有用户组都映射为匿名用户组账户，并指定该匿名用户组账户为本地用户组账户（GID=xxx） |

|    其他选项    |                   选项说明                   |
| :--------: | :--------------------------------------: |
|   secure   |  限制客户端只能从小于1024的tcp/ip端口连接nfs服务器（默认设置）   |
|  insecure  |        允许客户端从大于1024的tcp/ip端口连接服务器        |
|    sync    |     将数据同步写入内存缓冲区与磁盘中，效率低，但可以保证数据的一致性     |
|   async    |          将数据先保存在内存缓冲区中，必要时才写入磁盘          |
|   wdelay   | 检查是否有相关的写操作，如果有则将这些写操作一起执行，这样可以提高效率（默认设置） |
| no_wdelay  |          若有写操作则立即执行，应与sync配合使用           |
|  subtree   |   若输出目录是一个子目录，则nfs服务器将检查其父目录的权限(默认设置)    |
| no_subtree | 即使输出目录是一个子目录，nfs服务器也不检查其父目录的权限，这样可以提高效率  |

### 共享目录开放匿名访问权限
当客户端机器访问共享目录时，希望获取NFS服务端机器的某一用户权限，可以采用匿名访问的方式。

在服务器上创建用户组nfsanon和用户nfsanon，并将用户nfsanon添加到用户组nfsanon中。
```Shell
[root@www ~]# groupadd -g 45 nfsanon
[root@www ~]# useradd -u 45 -g nfsanon nfsanon
```
当然也可以把已有的用户添加进用户组nfsanon中
```Shell
[root@www ~]# gpasswd -a root nfsanon
[root@www ~]# id root
uid=0(root) gid=0(root) groups=45(nfsanon),0(root)
```

## NFS服务的启动与停止

在`/etc/exports`配置文件进行了正确的配置之后，就可以正常启动NFS服务了。

```Shell
systemctl start  nfs.service
systemctl start  rpcbing.service
```

### 查询NFS服务状态

nfs服务状态
```Shell
[nextradio_csg/audio/pgc_audio]# service nfs status
Redirecting to /bin/systemctl status  nfs.service
* nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; disabled; vendor preset: disabled)
   Active: active (exited) since Tue 2017-06-13 19:41:01 CST; 23h ago
  Process: 32586 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 32582 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 32586 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service

Jun 13 19:41:01 systemd[1]: Starting NFS server and services...
Jun 13 19:41:01 systemd[1]: Started NFS server and services.
```

rpcbind状态
```Shell
[nextradio_csg/audio/pgc_audio]# service rpcbind status       
Redirecting to /bin/systemctl status  rpcbind.service
* rpcbind.service - RPC bind service
   Loaded: loaded (/usr/lib/systemd/system/rpcbind.service; indirect; vendor preset: enabled)
   Active: active (running) since Tue 2017-06-13 19:30:48 CST; 23h ago
  Process: 29372 ExecStart=/sbin/rpcbind -w ${RPCBIND_ARGS} (code=exited, status=0/SUCCESS)
 Main PID: 29379 (rpcbind)
   CGroup: /system.slice/rpcbind.service
           `-29379 /sbin/rpcbind -w

Jun 13 19:30:48 systemd[1]: Starting RPC bind service...
Jun 13 19:30:48 systemd[1]: Started RPC bind service.
```

### 查看NFS服务开启的端口信息
```Shell
[nextradio_csg/audio/pgc_audio]# netstat -tunlp | grep 2049
tcp        0      0 0.0.0.0:2049            0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::2049                 :::*                    LISTEN      -                   
udp        0      0 0.0.0.0:2049            0.0.0.0:*                           -                   
udp6       0      0 :::2049                 :::*                                -      
```

```Shell
[nextradio_csg/audio/pgc_audio]# netstat -tunlp | grep rpc
tcp        0      0 0.0.0.0:59842           0.0.0.0:*               LISTEN      32579/rpc.statd     
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      29379/rpcbind       
tcp        0      0 0.0.0.0:20048           0.0.0.0:*               LISTEN      32580/rpc.mountd    
tcp6       0      0 :::42126                :::*                    LISTEN      32579/rpc.statd     
tcp6       0      0 :::111                  :::*                    LISTEN      29379/rpcbind       
tcp6       0      0 :::20048                :::*                    LISTEN      32580/rpc.mountd    
udp        0      0 0.0.0.0:20048           0.0.0.0:*                           32580/rpc.mountd    
udp        0      0 0.0.0.0:43818           0.0.0.0:*                           32579/rpc.statd     
udp        0      0 0.0.0.0:111             0.0.0.0:*                           29379/rpcbind       
udp        0      0 0.0.0.0:716             0.0.0.0:*                           29379/rpcbind       
udp        0      0 127.0.0.1:955           0.0.0.0:*                           32579/rpc.statd     
udp6       0      0 :::20048                :::*                                32580/rpc.mountd    
udp6       0      0 :::44230                :::*                                32579/rpc.statd     
udp6       0      0 :::111                  :::*                                29379/rpcbind       
udp6       0      0 :::716                  :::*                                29379/rpcbind
```

NFS服务默认启动的是**2049**端口，rpcbind默认启动的是**111**端口。

使用rpcinfo命令查看rpc相关信息，格式如下：

```Shell
rpc [option][IP|hostname]
```

其中option: -p 显示所有的port与programe信息

```
[nextradio_csg/audio/pgc_audio]# rpcinfo -p localhost
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100024    1   udp  43818  status
    100024    1   tcp  59842  status
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100003    4   udp   2049  nfs
    100227    3   udp   2049  nfs_acl
    100021    1   udp  41811  nlockmgr
    100021    3   udp  41811  nlockmgr
    100021    4   udp  41811  nlockmgr
    100021    1   tcp  28791  nlockmgr
    100021    3   tcp  28791  nlockmgr
    100021    4   tcp  28791  nlockmgr
```

### 停止NFS服务
只需要停止NFS服务即可，rpcbing服务可以不用停止
```
service nfs stop
```

### 设置NFS服务器的自动启动状态
```
**待补充**
```

# NFS客户端设置
NFS客户端的设置分两类：手动挂载NFS服务器的共享目录、设置开机自启动挂载NFS共享目录

## 手动挂载NFS服务器的共享目录

### 首先通过showmount指令查看NFS服务器的状态

```Shell
[cfs_files_migrate-1.0/bin]# showmount -e 192.168.1.123
Export list for 192.168.1.123:
/nextradio_csg 192.168.1.222
```

从结果可以看出，NFS服务器向 `192.168.1.222` 共享了 `/nextradio_csg` 目录。

### 在客户端机器挂载共享目录
```Shell
[cfs_files_migrate-1.0/bin]# mkdir /nextradio_csg
[cfs_files_migrate-1.0/bin]# mount -t nfs 192.168.1.123:/nextradio_csg /nextradio_csg
[cfs_files_migrate-1.0/bin]# df -H
Filesystem             Size   Used  Avail Use% Mounted on
/dev/sda1               11G   7.2G   3.0G  71% /
/dev/sda3               22G   3.3G    17G  17% /usr/local
/dev/sda4              261G   4.7G   243G   2% /data
tmpfs                   17G    25k    17G   1% /dev/shm
192.168.1.123:/nextradio_csg
                       2.2T   639G   1.5T  32% /nextradio_csg
```

通过 `df -H` 查看挂载结果。发现已经过了一块大小为2.2T的磁盘，命名为`nextradio_csg`

## 设置开机自启动挂载NFS共享目录

```
**待补充**
```

# NFS的常用命令
下面来介绍两个经常用到的查看命令
## showmount
```Shell
 格式：showmount [option] [IP|hostname]
        option:
            -a：显示当前主机与客户端的NFS连接共享的状态；
            -e：显示某台主机的/etc/exports所共享的目录信息。
```

## exportfs
```Shell
格式：exportfs [option]
  option:
            -a：全部挂载（或卸载）/etc/exports文件中的设置；
            -r：重新挂载/etc/exports中的设置；
            -u：卸载某一目录；
            -v：将命令输出显示到屏幕。
```

### 重新挂载一次 exports,不重启
```
[nextradio_csg/audio/pgc_audio]# exportfs -arv
exporting 192.168.1.223:/nextradio_csg
exporting 192.168.1.224:/nextradio_csg
exporting 192.168.1.225:/nextradio_csg
```

### 将已共享的NFS目录资源，全部卸载
```
[nextradio_csg/audio/pgc_audio]# exportfs -auv
```
