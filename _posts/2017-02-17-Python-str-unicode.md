---
layout: post
title: Python2之编码问题集锦
date: 2017-02-17 19:35:11.000000000 +09:00
updated: 2017-03-09 19:35:11.000000000 +09:00
comments: true
tags:
- Python
- 编码
categories:
- 编写高质量python代码
---

> `Python2.7`版本的编码问题，一直被开发者所诟病.该文章主要用于记录平时开发过程中所遇到的关于编码的问题.
> 以及关于**编码**问题的一些总结:正确进行编码转码?如何机智的绕开编码问题?

<!--more-->

## python的编码基础
### str与unicode
`str`和`unicode`都继承自`basestring`类.从构成上来讲,`str`是字符串,而`unicode`是由经过编码后的字节组成的序列.
但是,在Python2中,`unicode`才是真正意义上的字符串,任何形式单一的字符在`unicode`中的长度都是1.

``` python
str_utf_8 = b'腾'        # b标明强制转换字符串类型为str
print len(str_utf_8)     # 3
print repr(str_utf_8)    # '\xe8\x85\xbe'

uni_utf_8 = u'腾'        # u标明强制转换字符串类型为unicode
print len(uni_utf_8)     # 1
print repr(uni_utf_8)    # u'\u817e'
```

### str与unicode之间的相互转换

str与unicode之间的相互转换方法如下: 

|    源    |   目标    |  行为  |   方法   |
| :-----: | :-----: | :--: | :----: |
|   str   | unicode |  编码  | decode |
| unicode |   str   |  解码  | encode |



因为`str`和`unicode`都继承自`basestring`类,而且`decode`方法`unicode`方法都是放在`basestring`中的,
所以,如果方法用反了,Python是不会抛出异常的.对于新手来说,经常会在这里踩坑

### 字符编码声明
在脚本文件中,往往会遇到非ASCII字符,所以需要在脚本文件的头部进行**字符编码**的声明
``` python
#-*- coding: UTF-8 -*-
```
这里需要注意的是,声明的编码必须与文件实际保存的编码一直,否则很大几率会出现代码解析异常
 - 实际上Python只检查、coding和编码字符串,其他的字符都是为了美观加上的
 - Python中可用的字符编码有很多,并且还有许多别名,还不区分大小写,比如UTF-8可以写成u8
 - 参见[Standard Encodings][1]

#### 统一编码方式
``` python
from __future__ import unicode_literals
```
Python中有些库的接口要求参数必须是str类型字符串,有些接口要求参数必须是unicode类型字符串。
如上所阐述,同一个字符使用不同的编码格式,长度往往是不同的。
以str类型的字符串调用len()和遍历时,是以字节为单位,而unicode类型的字符串调用len()和遍历是以字符为单位。
另外,Django,Django REST framework的接口都是返回unicode类型的字符串。
为了统一,建议使用 `from __future__ import unicode_literals` 来统一模块中显式出现的所有字符串为unicode类型。

### 错误案例

#### datetime.strftime接口
datetime.strftime() 只接受str类型的字符串，不接受unicode类型的字符串,所以对于该接口的正确使用方式：
``` python
#coding:utf-8
from __future__ import unicode_literals
from datetime import datetime

now = datetime.now()
print now.strftime('%m月%d日 %H:%M'.encode('utf-8'))  # 指明str类型字符串
```

#### urlencode与urldecode
url地址中,经常会包含有中文,或者请求参数包含有中文,这时往往需要对其中的中文和‘/’做编码转换
##### urlencode 
urllib库里面有个urlencode函数，可以把key-value这样的键值对转换成我们想要的格式，返回的是a=1&b=2这样的字符串，比如：
``` python
>>> from urllib import urlencode
>>> data = {
...     'a': 'test',
...     'name': '魔兽'
... }
>>> print urlencode(data)
a=test&amp;name=%C4%A7%CA%DE
```
如果只想对一个字符串进行urlencode转换，怎么办？urllib提供另外一个函数：quote()
``` python
>>> from urllib import quote
>>> quote('魔兽')
'%C4%A7%CA%DE'
```

##### urldecode
当urlencode之后的字符串传递过来之后，接受完毕就要解码了——urldecode。urllib提供了unquote()这个函数，可没有urldecode()！
``` python
>>> from urllib import unquote
>>> unquote('%C4%A7%CA%DE')
'\xc4\xa7\xca\xde'
>>> print unquote('%C4%A7%CA%DE')
魔兽
```

#### 编解码时，出现 UnicodeError 异常
这跟`str.decode([encoding[, errors]])`、`str.encode([encoding[, errors]])`的异常处理有关系,[Python2.7对decode的定义][2]
> Decodes the string using the codec registered for encoding. encoding defaults to the default string encoding. errors may be given to set a different error handling scheme. The default is 'strict', meaning that encoding errors raise UnicodeError. Other possible values are 'ignore', 'replace' and any other name registered via codecs.register_error(), see section Codec Base Classes.

>当使用编解码器解码字符串时，encoding默认的字符串为`encoding`.当解码出现异常时,可以设置错误处理方案,默认为`strict`, 也就是在编码出现错误时会引发`UnicodeError`.

这里我们使用最多的异常处理方式是`replace`,可以有效的在出现异常时，用合适的字符来代替.
[编解码异常处理方法][3]如下所示:


| Value               | Meaning                                  |
| :------------------ | :--------------------------------------- |
| 'strict'            | Raise UnicodeError (or a subclass); this is the default. |
| 'ignore'            | Ignore the character and continue with the next. |
| 'replace'           | Replace with a suitable replacement character; Python will use the official U+FFFD REPLACEMENT CHARACTER for the built-in Unicode codecs on decoding and ‘?’ on encoding. |
| 'xmlcharrefreplace' | Replace with the appropriate XML character reference (only for encoding). |
| 'backslashreplace'  | Replace with backslashed escape sequences (only for encoding). |


[1]: http://docs.python.org/library/codecs.html#standard-encodings
[2]: http://devdocs.io/python~2.7/library/stdtypes#str.decode
[3]: http://devdocs.io/python~2.7/library/codecs#codec-base-classes

