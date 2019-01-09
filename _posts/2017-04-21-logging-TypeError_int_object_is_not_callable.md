---
layout: post
title: 【Exception】 logging - TypeError 'int' object is not callable
date: 2017-04-21 18:35:11.000000000 +09:00
updated: 2017-04-21 18:35:11.000000000 +09:00
comments: true
tags:
- 异常记录
categories:
- Python
---

脚本输出日志的时候，出现了`TypeError 'int' object is not callable`的异常，最开始以为是字符串格式化时的数据类型的问题，其实不然。
<!--more-->

## 异常描述

code
``` python
import logging
logging.basicConfig(level=logging.DEBUG,
                format='%(asctime)s %(filename)s[line:%(lineno)d] %(levelname)s %(message)s',
                datefmt='%a, %d %b %Y %H:%M:%S',
                filename='myapp.log',
                filemode='a')
logging.WARNING("""[dont_has_sub_category] %d %s %d""" % (album_id, origin_album_key, category))
logging.ERROR("""[sub_category_ch_list_empty] %d %s %d""" % (album_id, categoty_id_map_en[category], sub_category))
```

error
``` shell
Traceback (most recent call last):
  File "/home/user_00/bearboy/ximalaya/category_mapping_script.py", line 265, in
    logging.WARNING("""[dont_has_sub_category] %d %s %d""" % (album_id, origin_album_key, category))
TypeError: 'int' object is not callable
```

## 问题解决

一开始，一直以为是字符串格式化中变量的问题，可是定位了很久，没有发现任何有异常的地方。最后，查看[logging模块官网][0]
对`logging`模块做了很详细的描述
``` python
logging.debug(msg, *args, **kwargs)
logging.info(msg, *args, **kwargs)
logging.warning(msg, *args, **kwargs)
logging.error(msg, *args, **kwargs)
```

仔细看，这里用的logger方法是**全小写**，讲代码中的大写全部改为小写后，问题解决了。

## 具体原因

> 可是，这里写错了方法名，为什么会报`TypeError`的类型异常呢。
> 原因很简单，因为`logging.DEBUG`等是真实存在的，只不错它并不是一个方法，而是一个常量

### Logging Levels

|  Level   | Numeric value |
| :------: | :-----------: |
| CRITICAL |      50       |
|  ERROR   |      40       |
| WARNING  |      30       |
|   INFO   |      20       |
|  DEBUG   |      10       |
|  NOTSET  |       0       |

### 验证
```python
>>> import logging
>>> type(logging.DEBUG)
<type 'int'>
```
当在调用`logging.DEBUG`去打印日志的时候，因为这是一个整形常量，被用作了方法，所以抛出了`TypeError`的异常

[0]: https://docs.python.org/2/library/logging.html#logging.Logger.info
