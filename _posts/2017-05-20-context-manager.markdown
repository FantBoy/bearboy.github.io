---
layout: post
title: 高质量python代码之ContextManager上下文管理器
date: '2017-05-20 14:46'
description: 利用content manager构建上下文管理器
feature: images/tag-plugins/python.jpg
toc: true
tags:
  - ContextManager
categories:
  - 编写高质量python代码
---

开发过程中，往往需要对一些处理过程进行异常处理，并且想要尽可能的保证整个过程被完整执行，且如果在过程中出现了异常，也能准备无误的恢复到执行前。python2.5之后的版本引入了with语句，来进行上下文管理(Context Manager)。with语句适用于对资源进行访问的场景，确保不管在执行过程中是否发生异常，都会执行必要的"清理"操作，释放资源。比如文件使用后自动关闭、线程中锁的自动获取和释放、数据库连接的创建和关闭以及销毁


<!--more-->



## 上下文管理器定义

| 术语                          | 名称      | 定义                                       |
| --------------------------- | ------- | ---------------------------------------- |
| Context Management Protocol | 上下文管理协议 | 管理器必须实现 __enter__() 和 __exit__()方法       |
| Context Manager             | 上下文管理器  | 负责执行 with 语句块上下文中的进入与退出操作                |
| runtime context             | 运行时上下文  | __enter__() 方法在语句体执行之前进入运行时上下文，__exit__() 在语句体执行完后从运行时上下文退出 |
| with-body                   | 语句体     | with 语句包裹起来的代码块，在执行语句体之前会调用上下文管理器的 __enter__() 方法，执行完语句体之后会执行 __exit__() 方法 |

## 语法&工作原理
### with语句的语法格式

``` python
with context_expression [as target(s)]:
    with-body
```

其中的`context_expression`必须返回一个上下文管理器对象，但是这个对象并不会被赋值给`target(s)`。如果指定了as子句的话，会将上下文管理器的`__enter__()`方法的返回值赋值给`target(s)`。 target(s)可以是单个变量，也可以是一个元祖(不能是仅仅由“,”分隔的变量列表，必须加"()")。

python对一些内建函数进行了改进，加入了对上下文管理器的支持，比如文件操作类、线程锁等。

``` python
with open(r'somefileName') as somefile:
    for line in somefile:
        print line
        # ...more code
```

打开文件使用with上下文管理器，可以保证在with语句执行完成或者执行中出现任何异常的时候，都能及时有效地关闭文件句柄。跟传统的`try...finally...`的功能类似。

``` python
somefile = open(r'somefileName')
  try:
      for line in somefile:
          print line
          # ...more code
  finally:
      somefile.close()
```



**已经加入了上下文管理协议支持的常用模块是 open、threading、decimal 等**





## 自定义上下文管理器
 - context_manager.__enter__() ：进入上下文管理器的运行时上下文，在语句体执行前调用。with 语句将该方法的返回值赋值给 as 子句中的 target，如果指定了 as 子句的话
 - context_manager.__exit__(exc_type, exc_value, exc_traceback) ：退出与上下文管理器相关的运行时上下文，返回一个布尔值表示是否对发生的异常进行处理。参数表示引起退出操作的异常，如果退出时没有发生异常，则3个参数都为None。**如果发生异常，返回True 表示不处理异常**，否则会在退出该方法后重新抛出异常以由 with 语句之外的代码逻辑进行处理。如果该方法内部产生异常，则会取代由 statement-body 中语句产生的异常。要处理异常时，不要显示重新抛出异常，即不能重新抛出通过参数传递进来的异常，只需要将返回值设置为 False 就可以了。之后，上下文管理代码会检测是否 __exit__() 失败来处理异常.

假设有一个资源 DummyResource，这种资源需要在访问前先分配，使用完后再释放掉；分配操作可以放到 __enter__() 方法中，释放操作可以放到 __exit__() 方法中。

``` python
class DummyResource:
    def __init__(self, tag):
        self.tag = tag
        print 'Resource [%s]' % tag
    def __enter__(self):
        print '[Enter %s]: Allocate resource.' % self.tag
        return self	  # 可以返回不同的对象
    def __exit__(self, exc_type, exc_value, exc_tb):
        print '[Exit %s]: Free resource.' % self.tag
        if exc_tb is None:
            print '[Exit %s]: Exited without exception.' % self.tag
        else:
            print self, exc_type, exc_value, exc_tb
            print '[Exit %s]: Exited with exception raised.' % self.tag
            return False   # 可以省略，缺省的None也是被看做是False

with DummyResource('Normal'):
    print '[with-body] Run without exceptions.'

with DummyResource('With-Exception'):
    print '[with-body] Run with exception.'
    raise Exception
    print '[with-body] Run with exception. Failed to finish statement-body!'
```

Output
``` shell
Resource [Normal]
[Enter Normal]: Allocate resource.
[with-body] Run without exceptions.
[Exit Normal]: Free resource.
[Exit Normal]: Exited without exception.

Resource [With-Exception]
[Enter With-Exception]: Allocate resource.
[with-body] Run with exception.
[Exit With-Exception]: Free resource.
<__main__.DummyResource instance at 0x0000000002A85948> <type 'exceptions.Exception'>  <traceback object at 0x0000000002A859C8>
[Exit With-Exception]: Exited with exception raised.
Traceback (most recent call last):
  File "E:\github\MyMarkdownFiles\script\context_manager_demo.py", line 34, in <module>
    raise Exception
Exception
```


DummyResource 中的 __enter__() 返回的是自身的引用，这个引用可以赋值给 as 子句中的 target 变量；返回值的类型可以根据实际需要设置为不同的类型，不必是上下文管理器对象本身。

__exit__() 方法中对变量 exc_tb 进行检测，**如果不为 None，表示发生了异常**，返回 False 表示需要由外部代码逻辑对异常进行处理；注意到如果没有发生异常，缺省的返回值为 None，在布尔环境中也是被看做 False，但是由于没有异常发生，__exit__() 的三个参数都为 None，上下文管理代码可以检测这种情况，做正常处理。



可以自定义上下文管理器来对软件系统中的资源进行管理，比如数据库连接、共享资源的访问控制等。Python 在线文档 [Writing Context Managers][0] 提供了一个针对数据库连接进行管理的上下文管理器的简单范例。



## contextlib 模块

contextlib 模块提供了3个对象：装饰器 contextmanager、函数 nested 和上下文管理器 closing。使用这些对象，可以对已有的生成器函数或者对象进行包装，加入对上下文管理协议的支持，避免了专门编写上下文管理器来支持 with 语句。

### 装饰器 contextmanager
contextmanager 用于对生成器函数进行装饰，生成器函数被装饰以后，返回的是一个上下文管理器，其 __enter__() 和 __exit__() 方法由 contextmanager 负责提供，而不再是之前的迭代子。被装饰的生成器函数只能产生一个值，否则会导致异常 RuntimeError；产生的值会赋值给 as 子句中的 target，如果使用了 as 子句的话。下面看一个简单的例子。

``` python
from contextlib import contextmanager

@contextmanager
def demo():
    print '[Allocate resources]'
    print 'Code before yield-statement executes in __enter__'
    yield '*** contextmanager demo ***'
    print 'Code after yield-statement executes in __exit__'
    print '[Free resources]'

with demo() as value:
    print 'Assigned Value: %s' % value
```

Output
``` shell
[Allocate resources]
Code before yield-statement executes in __enter__
Assigned Value: *** contextmanager demo ***
Code after yield-statement executes in __exit__
[Free resources]
```

生成器函数中 yield 之前的语句在 __enter__() 方法中执行，yield 之后的语句在 __exit__() 中执行，而 yield 产生的值赋给了 as 子句中的 value 变量。

需要注意的是，contextmanager 只是省略了 __enter__() / __exit__() 的编写，但并不负责实现资源的“获取”和“清理”工作；“获取”操作需要定义在 yield 语句之前，“清理”操作需要定义 yield 语句之后，这样 with 语句在执行 __enter__() / __exit__() 方法时会执行这些语句以获取/释放资源，即生成器函数中需要实现必要的逻辑控制，包括资源访问出现错误时抛出适当的异常

### 函数 nested
> nested 可以将多个上下文管理器组织在一起，避免使用嵌套 with 语句。

``` python
with nested(A(), B(), C()) as (X, Y, Z):
    # with-body code here
```

执行顺序类似于

``` python
with A() as X:
    with B() as Y:
        with C() as Z:
            # with-body code here
```
**需要注意的是，发生异常后，如果某个上下文管理器的 __exit__() 方法对异常处理返回 False，更外层的上下文管理器也不会监测到异常**

### 上下文管理器 closing
上下文管理 closing 实现

``` python
class closing(object):
    # help doc here
    def __init__(self, thing):
        self.thing = thing
    def __enter__(self):
        return self.thing
    def __exit__(self, *exc_info):
        self.thing.close()
```

上下文管理器会将包装的对象赋值给 as 子句的 target 变量，同时保证打开的对象在 with-body 执行完后会关闭掉。closing 上下文管理器包装起来的对象必须提供 close() 方法的定义，否则执行时会报 AttributeError 错误。

自定义支持 closing 的对象

``` python
class ClosingDemo(object):
    def __init__(self):
        self.acquire()
    def acquire(self):
        print 'Acquire resources.'
    def free(self):
        print 'Clean up any resources acquired.'
    def close(self):
        self.free()

with closing(ClosingDemo()):
    print 'Using resources'
```



> closing 适用于提供了 close() 实现的对象，比如网络连接、数据库连接等，也可以在自定义类时通过接口 close() 来执行所需要的资源“清理”工作。





#### Mysql工具的上下文管理器实现
mysql操作往往需要实现"原子"操作,以及各种异常监控、处理和回滚等。

``` python

READ_RADIO_MYSQL_DB_CONF = {
    'host': '',
    'port': 3306,
    'user': '',
    'passwd': '',
    'db': '',
    'charset': 'utf8',
}


class MySQLDbTool:
    def __init__(self, db_conf):
        self.conf = db_conf
    def __enter__(self):
        self.crawler_conn = MySQLdb.connect(**self.conf)
        self.crawler_cursor = self.crawler_conn.cursor()
        return self.crawler_cursor
    def __exit__(self, exc_type, exc_value, exc_tb):
        if exc_tb is None:
            self.crawler_conn.commit() #提交所有操作
            self.crawler_cursor.close()
            self.crawler_conn.close()

        else:
            print self, exc_type, exc_value, exc_tb
            self.crawler_conn.rollback() #出现异常时回滚
            return False   # 可以省略，缺省的None也是被看做是False

```

[0]: https://docs.python.org/release/2.6/whatsnew/2.6.html?cm_mc_uid=41026434344714952813058&amp;amp;amp;amp;amp;amp;amp;amp;amp;amp;cm_mc_sid_50200000=1495281305#module-contextlib
