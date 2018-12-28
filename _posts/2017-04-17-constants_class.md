---
layout: post
title: 常量的管理类
date: 2017-04-17 17:00:00 +09:00
toc: true
description: 编写高质量python代码之常量的管理类
comments: true
tags:
- Python
categories:
- 编写高质量python代码
---

在多进程系统或者复杂的逻辑任务中，往往需要统一管理`constance`常量。而`Python`语言本身并没有提供const，所以就需要开发者自己来实现。

<!--more-->

## 定义_constance常量管理类

定义Constance.py

``` python
import sys

class _constance(object):
    class ConstError(TypeError): pass
    def __setattr__(self, key, value):
        """the constants key can be assignment once
        """
        if self.__dict__.has_key(key):
            raise self.ConstError, "const.key[%s] is exists!" % key
        else:
            self.__dict__[key] = value
    def __getattr__(self, key):
        if self.__dict__.has_key(key):
            return self.key
        else:
            return None

sys.modules[__name__] = _constance()  #__name__ = 'Constants'
```

 - 使用`sys.modules[name]`可以获取一个模块对象，并可以通过该对象获取模块的属性

 - 使用了`sys.modules`向系统字典中注入了一个`_constance对象`从而实现了在执行`import name`时实际获取了一个`_constance实例`的功能


`sys.module`在文档中的描述如下:
<p></p>
 - This is a dictionary that maps module names to modules which have already been loaded. This can be manipulated to force reloading of modules and other tricks. Note that removing a module from this dictionary is not the same as calling reload() on the corresponding module object.




## 常量类_constance的使用

`sys.modules[name] = _constance()`这条语句将系统已加载的模块列表中的const替换为了_constance(),即一个Const实例

``` python
import Constance

Constance.CFS_PATH_DIR = '/cfs_fm/audio/'

print Constance.CFS_PATH_DIR
>>> /cfs_fm/audio/

print Constance.FILEPATH
>>> None

Constance.CFS_PATH_DIR = '/cfs_fm/audio/new/'
>>> Traceback (most recent call last):
  File "E:\Bearboy_cloud_disk\docs\script_test\constances_class\test.py", line 9, in <module>
    Constance.CFS_PATH_DIR = '/cfs_fm/audio/new/'
  File "E:\Bearboy_cloud_disk\docs\script_test\constances_class\Constance.py", line 12, in __setattr__
    raise self.ConstError, "const.key[%s] is exists!" % key
Constance.ConstError: const.key[CFS_PATH_DIR] is exists!
```

## 补充说明import module和from module import的区别
  - import module只是将module的name加入到目标文件的局部字典中，不需要对module进行解释
  - from module import xxx需要将module解释后加载至内存中，再将相应部分加入目标文件的局部字典中
  - python模块中的代码仅在首次被import时被执行一次



> `import Constance`时，发生了`sys.modules[__name__] = _constance()`，此时，`Constance`模块已经加载进了